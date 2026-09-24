# Ansible unreachable hosts: ignore_unreachable, exit codes, meta end_host, wait_for_connection

## Context

A play set `ignore_unreachable: true` so that one SSH-dead host would not stop the others. That
raised several questions:

- does the dead host leave the play?
- what does PLAY RECAP count for it?
- which exit code does `ansible-playbook` return?
- does a `wait_for_connection` + `meta: end_host` barrier drop dead hosts cleanly?

The answers below come from reading ansible-core source at **v2.17.14** and comparing it with
**v2.20.2** (`repos/ansible`). The logic is the same in both unless a difference is called out.
Line numbers are for v2.17.14, with paths relative to `lib/ansible/`.

Every behavior marked "verified" was reproduced on ansible-core 2.17.14 (Python 3.10, OpenSSH
10.3) against three kinds of host: a local-connection host, SSH to `127.0.0.1:1` (connection
refused), and SSH to `192.0.2.1` (black hole). The ssh setting was `[ssh_connection] retries = 3`.

The key experiments were run again on ansible-core 2.20.2 (Python 3.12). Exit codes and recap
counters were identical; only the wording of some messages changed, as noted below.

## Finding

### 1. `ignore_unreachable` keeps the host in the play and counts the result as ok + ignored

- `ignore_unreachable` is an inheritable `FieldAttribute` on `Base` (`playbook/base.py:737`). It is
  inherited play -> block -> task.
- The implicit "Gathering Facts" task inherits it too, because `PlayIterator` builds that task
  under `Block(play=self._play)` (`executor/play_iterator.py:152-156`). Verified: gathering prints
  `UNREACHABLE! ... ...ignoring`.
- When a result is unreachable and ignored (`plugins/strategy/__init__.py:624-633`):
  - the host is **not** added to `_tqm._unreachable_hosts` or to `play._removed_hosts`;
  - `dark` is not incremented, so the recap shows `unreachable=0` even for a host that was
    unreachable on every task;
  - `ok` and `ignored` are each incremented instead.
- The host is not marked failed, so every later task tries SSH again and pays the full ssh retry
  cycle each time.
- An unreachable result returns from the task right away (`executor/task_executor.py:647-648`).
  `changed_when`, `failed_when` and `until`/`retries` are never evaluated, so `retries` does not
  retry an unreachable host. Verified.
- Tasks that never connect to the host still run for it: `delegate_to: localhost`, `debug`,
  `set_fact`, `meta`. Those are the tasks where such a host actually fails later.
- Worked example (verified). The run produced `ok=9 changed=0 unreachable=0 failed=1 ignored=8`
  and exit code 2:
  - implicit gathering plus 7 remote tasks were unreachable: 8 x (`ok`+1, `ignored`+1);
  - one `delegate_to: localhost` task succeeded: `ok=9`;
  - one delegated task templated a fact that was never gathered and failed: `failed=1`.

### 2. Registered results after an ignored unreachable or a skip

- Unreachable (2.17): `{"unreachable": true, "msg": "...", "changed": false}`. There is no
  `failed`, `rc` or `stdout` key.
- Unreachable (2.20): the same keys plus `"exception": "(traceback unavailable)"`, and `msg` starts
  with `Task failed: `. Still no `failed` key (`errors/__init__.py`
  `AnsibleConnectionFailure.omit_failed_key`). Verified.
- Skipped by `when`: `{"changed": false, "skipped": true, "skip_reason": "Conditional result was
  False", "false_condition": <condition>}` (`executor/task_executor.py:479-483`).
- The status tests (`plugins/test/core.py:42-90`):

  | Test | What it returns |
  |------|-----------------|
  | `failed` | `result.failed` |
  | `succeeded` | not `failed` |
  | `unreachable` | `result.unreachable` |
  | `reachable` | not `unreachable` |
  | `skipped` | `result.skipped` |

- **Trap:** on an unreachable result, `is failed` is False and `is succeeded` is True. Test it
  with `is unreachable`. Verified.
- **Trap:** a later `when: "'x' in reg.stdout"` on an unreachable or skipped result fails that
  task with `error while evaluating conditional (...): 'dict object' has no attribute 'stdout'`.
  `ignore_unreachable` does not cover this. Verified.
- A delegated task templates its module args with the **original** host's variables. Validation
  happens at `executor/task_executor.py:533`, before the switch to the delegated host's variables
  at `:561`. So a fact that was never gathered on host H makes the task fail. That is a normal task
  failure, not an unreachable result, whatever `ignore_unreachable` says. Verified. The message
  differs by version:
  - 2.17: `The task includes an option with an undefined variable` (`playbook/base.py:565-571`).
  - 2.20: `Task failed: Finalization of task args for '<module>' failed: Error while resolving value
    for '<arg>': '<var>' is undefined`.

### 3. Exit codes

- The task queue manager's codes (`executor/task_queue_manager.py:124-129`):

  | Code | Meaning |
  |------|---------|
  | 0 | OK |
  | 1 | error |
  | 2 | failed hosts |
  | 4 | unreachable hosts |
  | 8 | break play (internal only) |
  | 255 | unknown error |

- Each play picks one code (`plugins/strategy/__init__.py:325-332`), checking in this order:
  1. any non-OK result the strategy already returned;
  2. 4 if any host is in `_unreachable_hosts`;
  3. 2 if any host failed;
  4. otherwise 0.

  **So 4 masks 2.** Verified: one dead host plus one failed host gives exit 4.
- Only unreachables that were **not** ignored go into `_unreachable_hosts`. With
  `ignore_unreachable` on, the same pair of hosts gives exit 2. Verified.
- A `max_fail_percentage` break returns 8, which is turned into 2
  (`executor/playbook_executor.py:194-195`). Because the strategy's own result is checked first,
  that 2 wins over 4.
- `ansible-playbook` returns the code of the last play it ran (`playbook_executor.py:188`,
  `:261`/`:270`). Failed and unreachable hosts are carried into later plays
  (`task_queue_manager.py:328-334`, `playbook_executor.py:172`), so they keep affecting the final
  code.
- The CLI adds its own codes (`cli/__init__.py:655-698`):

  | Code | Meaning |
  |------|---------|
  | 1 | `AnsibleError` |
  | 4 | **also** `AnsibleParserError` |
  | 5 | bad CLI options |
  | 99 | Ctrl-C |
  | 250 | unexpected exception |

  Exit 4 is therefore ambiguous. In 2.20, `ExitCode.PARSER_ERROR = 4` carries a FIXME about the
  clash with `HOST_UNREACHABLE`.

### 4. `meta: end_host`

- Added in Ansible 2.8 (`modules/meta.py:33`).
- It sets the host's run state to COMPLETE and appends the host to `play._removed_hosts`
  (`plugins/strategy/__init__.py:1016-1025`). The host is neither failed nor unreachable, so the
  exit code is unaffected. It also disappears from `ansible_play_hosts` and `ansible_play_batch`
  (`vars/manager.py:502-503`).
- The effect is limited to the current play. Every play runs on a fresh `Play.copy()`, whose
  `_removed_hosts` list is empty, and only failed and unreachable hosts are carried forward. So the
  host comes back in the next play of the same run. Verified.
- It increments no stats counter, because meta results bypass `_process_pending_results`
  (`:1079-1093`). A host whose only activity was `end_host` does not appear in PLAY RECAP at all.
  Verified.
- It honors `when`, and the condition is evaluated for each host: the linear strategy does not
  run `end_host` just once (`plugins/strategy/linear.py:165`).
- If every host hits `end_host`, the play simply finishes. There is no "no hosts left" error, later
  plays still run, and the exit code is 0. Verified.
- **Trap:** a meta task's `when` is evaluated in the controller's strategy loop, not in a worker
  (`strategy/__init__.py:929-935`; `linear.py:358` only catches IOError/EOFError). If the
  condition raises an error, such as an undefined variable, the whole run aborts with exit 1 and no
  recap. 2.17 prints `ERROR!`; 2.20 prints `[ERROR]: Error while evaluating conditional`. Guard the
  condition with `is defined`. Verified on both.
- In 2.20 it goes through `PlayIterator.end_host()`, which also clears the fail state when the
  host is ended from inside a `rescue` section.

### 5. `wait_for_connection`

- Defaults (`plugins/action/wait_for_connection.py:39-42`): `timeout` 600, `delay` 0, `sleep` 1,
  `connect_timeout` 5.
- A host that never answers produces **failed**, not unreachable (`:109-111`), with the message
  `timed out waiting for ping module test: <last ssh error>`. `ignore_unreachable` does not cover
  it; use `ignore_errors: true` plus `register`. Verified.
- No connection is attempted before the poll loop. `TRANSFERS_FILES = False` (`:36`), so
  `ActionBase.run` does not create the remote tmpdir early (`plugins/action/__init__.py:129-130`).
  Any exception inside an attempt is caught and the attempt is retried (`:54`).
- `connect_timeout` is only passed to a connection plugin's `transport_test`, and no ansible-core
  connection plugin implements one (`:78-94`, `:103`). **For ssh it has no effect.**
- For ssh, the time allowed per connection is the ssh `timeout` option: default 10, sent as
  `-o ConnectTimeout=10` (`plugins/connection/ssh.py:333-353`, `:774-779`).
- Each attempt is a full ssh call subject to `reconnection_retries` (`[ssh_connection] retries`).
  That means `retries + 1` tries, with pauses of `2**n - 1` seconds between them (0, 1, 3, 7, 15,
  capped at 30), and only ssh exit code 255 is retried (`ssh.py:496-555`).
- The timeout is only checked between attempts, so the task can run past it by up to one full
  attempt:
  - Verified: `timeout: 5`, ConnectTimeout 2, retries 3, black-holed IP: elapsed 13 s.
  - With ConnectTimeout 10 and retries 3, one attempt against a black hole takes about
    4 x 10 + 4 = 44 s.
- Inferred from code, not reproduced: the tmpdir cleanup after the loop (`:117`) sits outside the
  `try`. Suppose one attempt managed to create a remote tmpdir (`plugins/action/__init__.py:1028-1030`
  reuses it) and the host then went away. The task can then return **unreachable** instead of
  failed. A barrier's condition should therefore be `is failed or is unreachable`.
- **Trap:** `failed_when: false` on the barrier sets `failed` to false, so `when: barrier is failed`
  never fires. The `msg` key is still there. Use `ignore_errors: true` instead. Verified.

### 6. Fact gathering around a barrier

- Implicit gathering runs once, at play start (`executor/play_iterator.py:277-300`).
- If it hits an ignored unreachable, the host carries on with no facts. Nothing gathers them
  again, even after the host becomes reachable. Verified.
- With the default `gathering = implicit`, the fact cache is not consulted.
- Put the pieces in this order:
  1. `gather_facts: false`;
  2. `wait_for_connection` with `ignore_errors: true` and `register`;
  3. `meta: end_host` with `when: barrier is failed or barrier is unreachable`;
  4. an explicit `ansible.builtin.setup` or `gather_facts` task.

### 7. Alternatives and what they really do

- `meta: clear_host_errors` (added in 2.1) runs once for the play (`linear.py:165-166`):
  - it removes **all** play hosts from the failed and unreachable lists and clears their fail
    state (`strategy/__init__.py:985-994`, `play_iterator.py:546-557`);
  - a host that was unreachable, with ignore_unreachable off, comes back into the play. Once the
    other hosts finish, it runs the tasks it missed. Verified: exit 0, with `unreachable=1` in the
    recap;
  - it brings hosts back; it does not drop them.
- `meta: end_batch` (2.12) and `meta: end_play` (2.2) apply to every host. They run once, and the
  first host's `when` decides. Without `serial`, `end_batch` behaves like `end_play`.
- `max_fail_percentage` counts only failed hosts, not unreachable ones (`linear.py:339`).
- `any_errors_fatal` is tripped by any unreachable result, **including ignored ones**
  (`linear.py:322-332`). Every remaining host is then marked failed without any stats increment,
  so the recap shows `failed=0` everywhere and the exit code is 2. Verified.
- No `ansible.cfg` setting changes how unreachable hosts are handled:
  - `ANY_ERRORS_FATAL` only sets the default for the keyword;
  - ssh `retries` and `timeout` only change how long it takes to detect the failure.
- Two built-in patterns make an unreachable host leave the play without exit 4. Both give exit 0
  (verified):
  - Single probe: `ignore_unreachable`, register a cheap task such as `ping`, then `meta: end_host`
    with `when: probe is unreachable`.
  - Polling barrier: the `wait_for_connection` pattern from section 6.

### 8. Cost under the linear strategy

- The linear strategy queues a task on every host and waits for all results before the next task
  (`linear.py:126-216`, `strategy/__init__.py:800-821`). A barrier therefore costs the slowest
  host's full timeout. Verified: the healthy host waited the full 5 s.
- There are `min(forks, batch size)` workers (`task_queue_manager.py:317`), and `_queue_task`
  blocks until one is free (`strategy/__init__.py:397-428`). When more hosts are dead than there
  are forks, they are processed in waves.
  - Verified with 2 dead hosts and `timeout: 5`: forks=50 took 6 s, forks=1 took 11 s.

## Source

- ansible-core source, tag `v2.17.14` (github.com/ansible/ansible), compared with `v2.20.2` in
  `repos/ansible`. Main v2.20.2 locations:
  - `plugins/strategy/__init__.py` 599-608 (unreachable branch), 305-313 (exit code),
    998-1006 (end_host), 967-994 (clear_host_errors, end_batch, end_play);
  - `executor/play_iterator.py` 674 (`end_host()`);
  - `playbook/base.py` 705 (`ignore_unreachable`);
  - `executor/task_executor.py` 427 (exception to result);
  - `errors/__init__.py` 23-32 (`ExitCode`), 238-251 (`AnsibleConnectionFailure`);
  - `plugins/action/wait_for_connection.py`: identical logic.
- Experiments: discovered via experimentation on ansible-core 2.17.14 / Python 3.10.19 /
  OpenSSH 10.3p1, using the hosts and settings described in Context.
