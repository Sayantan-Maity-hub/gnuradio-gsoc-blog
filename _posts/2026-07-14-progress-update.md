# GSoC Weekly Blog: Experiment State Tracking and SSH Improvements

This week I focused on two major areas of the Hardware-in-the-Loop (HIL) CI framework:

- Implementing the experiment state tracking mechanism.
- Resolving all code review issues raised by my mentor, Marcus Müller.

## Experiment State Tracking

At the beginning of the week, my understanding of experiment state tracking was different from what was actually required. My initial implementation attempted to determine the current execution stage by analyzing the user script and displaying which step of the script was currently running.

After discussing this with my mentor, I realized that this was not the intended approach. The state tracking should represent the state of the HIL framework itself rather than the execution flow of the user's script.

Based on this feedback, I redesigned the implementation to track the internal states of the execution framework. Instead of trying to interpret arbitrary user scripts, the framework now reports its own progress throughout the experiment lifecycle. This makes the implementation more reliable, easier to maintain, and independent of user-written code.

## Addressing Code Review Feedback

I also completed all of the issues raised during code review by Marcus Müller.

### Completed Improvements

- Consistently formatted the Python codebase.
- Executed uploaded scripts directly after making them executable instead of invoking them through `bash`.
- Properly quoted file paths when constructing shell commands to support paths containing spaces.
- Removed the hardcoded SSH credential location and made the SSH configuration more flexible.
- Updated `ssh.run_on_node()` to match the agreed-upon interface and behavior.
- Removed unnecessary `__pycache__` files from the repository.

These improvements make the project cleaner, more portable, and easier to maintain.

## Debugging the Node SSH Authentication

One of the most interesting challenges this week was fixing SSH connections to the CortexLab nodes.

Initially, I implemented node connections using Paramiko directly through the gateway (gateway → node). However, every connection attempt resulted in an `AuthenticationException`.

Since the following command worked successfully,

```bash
ssh -p 2222 root@mnode
```

I temporarily used the OpenSSH client to keep development moving while investigating why Paramiko was failing.

After several rounds of debugging, I discovered that the issue was related to the authentication method used by the CortexLab nodes.

The nodes accept **"none" authentication** instead of public-key or password authentication. OpenSSH automatically negotiates this authentication method during the SSH handshake, which is why the command-line `ssh` client connects successfully without requiring a password or key.

On the other hand, `SSHClient.connect()` in Paramiko automatically attempts public-key or password authentication, causing it to fail with an `AuthenticationException`.

The solution was to bypass `SSHClient.connect()` for the node connection and instead:

1. Create a `Transport` over the forwarded channel.
2. Call `start_client()`.
3. Authenticate explicitly using:

```python
transport.auth_none("root")
```

4. Attach the authenticated transport to an `SSHClient`.

This approach allows Paramiko to use the same authentication mechanism that OpenSSH negotiates automatically, without spawning an external `ssh` process.

As a result, the implementation is now fully Paramiko-based while preserving the authentication behavior expected by the CortexLab nodes.
---

**GitHub Issues Resolved This Week**

- ✅ Consistently format Python code (#6)
- ✅ Run script that you made executable, not bash (#5)
- ✅ Need to quote paths if running arguments through strings (#4)
- ✅ Can't hardcode credential location (#3)
- ✅ `ssh.run_on_node` does not fulfill agreed-upon methods (#2)
- ✅ Don't include `__pycache__` files
