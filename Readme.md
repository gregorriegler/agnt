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
Clear all agent registrations (asks for confirmation).

```bash
agnt start
```

#### `agnt rename <name>`
Register the current tmux pane as an agent with the specified name.

```bash
agnt rename alice
```

#### `agnt whoami`
Display the current agent name (or 'nobody' if not registered).

```bash
agnt whoami
# Output: alice
```

#### `agnt list`
List all registered agents across all tmux panes.

```bash
agnt list
# Output:
# alice
# bob
```

#### `agnt send <agent> <message>`
Send a message to another agent. The message will be prefixed with your agent name and sent to the target pane.

```bash
agnt send bob "Hello from Alice!"
```

The target agent will receive:
```
#[alice] Hello from Alice!
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

- **Lock File**: Uses `.agnt.lock` to track pane ID to agent name mappings
- **Message Format**: Messages are prefixed with `#[sender_name]` so recipients know who sent them
- **Stale Cleanup**: Automatically removes registrations for closed panes
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
- Use `agnt list` to see registered agents
- The target agent may have been closed; try `agnt list` to verify

**Stale registrations**
- Run `agnt list` which automatically cleans up closed panes
- Or use `agnt start` to clear all registrations

## License

This is free and unencumbered software released into the public domain.
