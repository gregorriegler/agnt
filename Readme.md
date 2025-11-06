# agnt - Agent Management Script

A bash script for managing named agents in tmux panes, enabling inter-agent communication.
Inspired by Lada Kesseler's talk: https://www.youtube.com/watch?v=_LSK2bVf0Lc&t=8125

## Overview

`agnt` allows you to register tmux panes as named agents and send messages between them. This is particularly useful for orchestrating multiple AI agents (like Claude) or creating multi-agent systems where different processes need to communicate.

## Installation

1. Make sure the script is executable:
```bash
chmod +x agnt
```

2. Optionally, add it to your PATH:
```bash
# Add to your ~/.bashrc or ~/.zshrc
export PATH="$PATH:/path/to/agents"
```

## Requirements

- tmux (Terminal Multiplexer)
- bash

## Usage

### Commands

#### `agnt start`
Clear all agent registrations in the current directory (asks for confirmation). Creates an empty `.agnt.lock` file if one doesn't exist. Always operates on the current directory's lock file.

```bash
agnt start
```

#### `agnt rename <name>`
Register the current tmux pane as an agent with the specified name.

```bash
agnt rename alice
```

**Agent Name Restrictions:**
- Cannot be empty
- Cannot contain `:` (colon) - used as delimiter in lock file format
- Cannot contain newlines
- Most other characters are supported, including spaces, `#`, `|`, `/`, `$`, etc.

#### `agnt whoami`
Display the current agent name (or 'nobody' if not registered).

```bash
agnt whoami
# Output: alice
```

#### `agnt list`
List all registered agents that are currently running the `claude` command. Only shows agents that are both registered and actively running Claude.

```bash
agnt list
# Output:
# alice
# bob
```

#### `agnt send <agent> <message>`
Send a message to another agent. The message will be prefixed with your agent name and sent to the target pane. Fails with an error if the target agent is not running the `claude` command.

```bash
agnt send bob "Hello from Alice!"
# Output: Message sent to bob

agnt send charlie "Hi there"
# Error: Agent 'charlie' is not running claude (running 'zsh')
```

The target agent will receive:
```
#[alice] Hello from Alice!
```

#### `agnt purge`
Remove all registered agents that are not currently running the `claude` command. This cleans up the lock file by removing stale entries and panes that have switched to other commands.

```bash
agnt purge
# Output:
# Removing 'alice' (pane %1 running 'zsh', not 'claude')
# Removing 'bob' (pane %2 no longer exists)
#
# Purge complete: 2 agent(s) kept, 2 agent(s) removed
```

#### `agnt watch [interval]`
Continuously monitor and display the list of active agents, refreshing every N seconds (default: 3). Press Ctrl+C to stop.

```bash
agnt watch        # refresh every 3 seconds
agnt watch 5      # refresh every 5 seconds
```

#### `agnt help`
Show help information.

```bash
agnt help
```

## Example: Two Claude Agents Counting from 1 to 10

This example demonstrates two Claude agents cooperating to count from 1 to 10, taking turns.

### Setup

1. Start a tmux session:
```bash
tmux new-session -s agents
```

2. Split the window vertically:
```
Ctrl+b %
```

3. In the left pane, initialize agent "counter1":
```bash
agnt rename counter1
claude
```

4. In the right pane, initialize agent "counter2":
```bash
agnt rename counter2
claude
```

### Agent Instructions

**In the counter1 pane:**
```
You are counter1. Your job is to count odd numbers from 1 to 9.

When you receive a message from counter2 with an even number, respond by sending the next odd number using:
agnt send counter2 <number>

Start by sending the number 1 to counter2.

Rules:
- Only send the next odd number after receiving an even number
- Stop after sending 9
- Keep your responses minimal - just acknowledge and send the next number
```

**In the counter2 pane:**
```
You are counter2. Your job is to count even numbers from 2 to 10.

When you receive a message from counter1 with an odd number, respond by sending the next even number using:
agnt send counter1 <number>

Wait for counter1 to start.

Rules:
- Only send the next even number after receiving an odd number
- Stop after sending 10
- Keep your responses minimal - just acknowledge and send the next number
```

### Expected Output

**counter1 pane:**
```
> agnt send counter2 1
Message sent to counter2

#[counter2] 2

> agnt send counter2 3
Message sent to counter2

#[counter2] 4

> agnt send counter2 5
Message sent to counter2

#[counter2] 6

> agnt send counter2 7
Message sent to counter2

#[counter2] 8

> agnt send counter2 9
Message sent to counter2

#[counter2] 10
```

**counter2 pane:**
```
#[counter1] 1

> agnt send counter1 2
Message sent to counter1

#[counter1] 3

> agnt send counter1 4
Message sent to counter1

#[counter1] 5

> agnt send counter1 6
Message sent to counter1

#[counter1] 7

> agnt send counter1 8
Message sent to counter1

#[counter1] 9

> agnt send counter1 10
Message sent to counter1
```

## How It Works

- **Lock File**: Uses `.agnt.lock` to track pane ID to agent name mappings. When reading the lock file, `agnt` searches upward through parent directories (within `$HOME`) to find the nearest lock file, allowing you to manage agents project-wide.
- **Claude Process Verification**: The `list` and `send` commands verify that the target pane is running the `claude` command, ensuring messages only go to active Claude agents.
- **Message Format**: Messages are prefixed with `#[sender_name]` so recipients know who sent them
- **Automatic Cleanup**: `purge` command removes agents not running `claude`, and `list` filters out inactive agents automatically
- **tmux Integration**: Uses `tmux send-keys` to deliver messages directly to target panes

## Advanced Usage

### Multi-Agent Workflows

You can create complex multi-agent systems by:

1. Creating multiple tmux panes
2. Registering each as a different agent
3. Having each agent listen for specific message patterns
4. Orchestrating communication between agents

### Integration with AI Assistants

The script works particularly well with AI assistants like Claude that can:
- Execute bash commands
- Parse incoming messages
- Make decisions about when and what to send
- Follow conversation protocols

## Troubleshooting

**"Error: Not in a tmux session"**
- Make sure you're running commands from within a tmux session

**"Error: Agent 'name' not found"**
- Use `agnt list` to see currently active agents
- The target agent may have been closed or switched to a different command
- Try `agnt watch` to monitor agents in real-time

**"Error: Agent 'name' is not running claude"**
- The agent pane exists but is not currently running the `claude` command
- Switch to that pane and start `claude`, or use `agnt purge` to clean up inactive agents

**"Error: agent name cannot contain ':' character"**
- Colons are used as delimiters in the lock file format and cannot be used in agent names
- Choose a different name without colons

**Agent not showing in `list` but is registered**
- `agnt list` only shows agents actively running `claude`
- If you've exited Claude or switched to another command, the agent won't appear
- Use `agnt purge` to clean up agents not running Claude
- Or use `agnt start` in the original directory to clear all registrations

**Multiple lock files in different directories**
- `agnt` searches upward from the current directory to find `.agnt.lock` (stopping at `$HOME`)
- This allows project-wide agent management
- Use `agnt start` in a specific directory to create a new lock file scope there

**Wrong sender name appearing in messages**
- This issue has been fixed in recent versions
- The script now uses fixed-string matching to prevent special characters from being misinterpreted
- Duplicate entries in the lock file are handled by taking only the first match

## License

Copyright 2025 David Tanzer (business@davidtanzer.net)

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.

