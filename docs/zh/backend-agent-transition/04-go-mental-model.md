# 用 Go 的脑子读这个仓库：核心结构体与伪代码手册

## 一、先把 Python 变量翻译成你熟悉的东西

如果你平时主要写 Go，那读这个仓库时不要被 Python 语法绊住。先把常见元素翻译一下：

| Python 写法 | 你可以脑补成 Go 里的什么 |
|---|---|
| `dict` | `struct` 或 `map[string]any` |
| `list` | `[]T` |
| `class` | `struct + method` |
| `Path` | `string` 或文件路径封装 |
| `lambda` | 闭包函数 |
| `threading.Thread` | goroutine |
| `Queue` | channel 或线程安全队列 |

---

## 二、这个仓库最核心的几个结构体

下面不是源码逐字翻译，而是帮助你建立心智模型的 Go 风格定义。

### 1. 对话消息

```go
type Message struct {
    Role    string      `json:"role"`    // user / assistant
    Content interface{} `json:"content"` // string 或 []ToolResult
}
```

### 2. 工具调用与结果

```go
type ToolCall struct {
    ID    string                 `json:"id"`
    Name  string                 `json:"name"`
    Input map[string]interface{} `json:"input"`
}

type ToolResult struct {
    Type      string `json:"type"`       // 固定是 tool_result
    ToolUseID string `json:"tool_use_id"`
    Content   string `json:"content"`
}
```

### 3. Todo 项

```go
type TodoItem struct {
    Content    string `json:"content"`
    Status     string `json:"status"`      // pending / in_progress / completed
    ActiveForm string `json:"activeForm"`  // 当前正在做的动作描述
}
```

### 4. 任务

```go
type Task struct {
    ID          int      `json:"id"`
    Subject     string   `json:"subject"`
    Description string   `json:"description"`
    Status      string   `json:"status"`
    Owner       string   `json:"owner"`
    Worktree    string   `json:"worktree"`
    BlockedBy   []int    `json:"blockedBy"`
    Blocks      []int    `json:"blocks"`
    CreatedAt   float64  `json:"created_at,omitempty"`
    UpdatedAt   float64  `json:"updated_at,omitempty"`
}
```

### 5. 队友消息

```go
type TeamMessage struct {
    Type      string                 `json:"type"`
    From      string                 `json:"from"`
    Content   string                 `json:"content"`
    Timestamp float64                `json:"timestamp"`
    Extra     map[string]interface{} `json:"-"`
}
```

### 6. Worktree

```go
type Worktree struct {
    Name      string  `json:"name"`
    Path      string  `json:"path"`
    Branch    string  `json:"branch"`
    TaskID    int     `json:"task_id"`
    Status    string  `json:"status"`
    CreatedAt float64 `json:"created_at"`
}
```

---

## 三、最小 agent loop，用 Go 伪代码怎么表示

这是全仓库最重要的一段逻辑。

```go
func AgentLoop(messages []Message, llm LLMClient, registry ToolRegistry) ([]Message, error) {
    for {
        resp, err := llm.Create(messages, registry.Definitions())
        if err != nil {
            return messages, err
        }

        messages = append(messages, Message{
            Role:    "assistant",
            Content: resp.Content,
        })

        if resp.StopReason != "tool_use" {
            return messages, nil
        }

        results := make([]ToolResult, 0)
        for _, call := range resp.ToolCalls {
            output, err := registry.Exec(call.Name, call.Input)
            if err != nil {
                output = "error: " + err.Error()
            }

            results = append(results, ToolResult{
                Type:      "tool_result",
                ToolUseID: call.ID,
                Content:   output,
            })
        }

        messages = append(messages, Message{
            Role:    "user",
            Content: results,
        })
    }
}
```

### 你应该看到的重点

1. 主循环不关心具体业务。
2. 业务能力都挂在工具注册表上。
3. 只要模型继续调用工具，循环就继续。

这就是“智能决策”和“工程执行”的边界。

---

## 四、TodoManager 用 Go 怎么理解

在 Python 里它是一个 class，在 Go 里你完全可以把它想成：

```go
type TodoManager struct {
    Items []TodoItem
}

func (m *TodoManager) Update(items []TodoItem) error {
    inProgress := 0
    for _, item := range items {
        if item.Content == "" {
            return errors.New("content required")
        }
        switch item.Status {
        case "pending", "in_progress", "completed":
        default:
            return errors.New("invalid status")
        }
        if item.Status == "in_progress" {
            inProgress++
        }
    }
    if inProgress > 1 {
        return errors.New("only one in_progress allowed")
    }
    m.Items = items
    return nil
}
```

### 它的业务意义

它不是做持久化，而是在主会话内维护一个轻量级执行计划。

### 你可以这样记

- Todo 是会话级状态
- Task 是系统级状态

这两个不要混。

---

## 五、TaskManager 用 Go 怎么理解

`s07` 之后，任务被放到磁盘文件里。用 Go 脑图来看，这像一个很朴素的文件型仓储层：

```go
type TaskStore interface {
    Create(subject, description string) (Task, error)
    Get(id int) (Task, error)
    Update(task Task) error
    List() ([]Task, error)
}
```

一个简化版流程如下：

```go
func CompleteTask(store TaskStore, taskID int) error {
    task, err := store.Get(taskID)
    if err != nil {
        return err
    }

    task.Status = "completed"
    if err := store.Update(task); err != nil {
        return err
    }

    tasks, _ := store.List()
    for _, t := range tasks {
        if contains(t.BlockedBy, taskID) {
            t.BlockedBy = removeInt(t.BlockedBy, taskID)
            _ = store.Update(t)
        }
    }
    return nil
}
```

### 这里最重要的点

任务不是“为了展示”，而是为了让系统具备真正的跨轮次状态。

---

## 六、BackgroundManager 用 Go 怎么理解

这块你可以直接脑补成 goroutine + channel。

```go
type BackgroundTask struct {
    ID      string
    Command string
    Status  string
    Result  string
}

type Notification struct {
    TaskID string
    Status string
    Result string
}

type BackgroundManager struct {
    Tasks map[string]*BackgroundTask
    Queue chan Notification
}
```

启动后台任务：

```go
func (m *BackgroundManager) Run(command string) string {
    id := NewID()
    m.Tasks[id] = &BackgroundTask{
        ID:      id,
        Command: command,
        Status:  "running",
    }

    go func() {
        result, err := Exec(command)
        status := "completed"
        if err != nil {
            status = "error"
            result = err.Error()
        }

        m.Tasks[id].Status = status
        m.Tasks[id].Result = result
        m.Queue <- Notification{
            TaskID: id,
            Status: status,
            Result: result,
        }
    }()

    return id
}
```

### 业务意义

agent 主循环不需要傻等长任务结束，而是可以继续思考下一步。

---

## 七、MessageBus 和队友系统，用 Go 怎么理解

你可以把 `.team/inbox/*.jsonl` 看成最简版消息队列。

```go
type MessageBus interface {
    Send(to string, msg TeamMessage) error
    ReadInbox(name string) ([]TeamMessage, error)
}
```

其中 `ReadInbox` 的关键不是“读”，而是“读完清空”，这意味着它是 drain-on-read 模式。

```go
func WorkerLoop(name string, bus MessageBus) {
    for {
        msgs, _ := bus.ReadInbox(name)
        for _, msg := range msgs {
            HandleMessage(msg)
        }

        // 然后继续做自己的 agent loop
        // 无活可做时进入 idle
    }
}
```

### 为什么这块很重要

因为一旦进入多 agent 协作，系统的复杂度就从“函数调用”升级为“组织通信”。

---

## 八、协议层，用 Go 怎么理解

`s10` 里很关键的是 `request_id`。这对后端开发者应该非常熟悉。

你可以把它想成：

```go
type ApprovalRequest struct {
    RequestID string
    From      string
    To        string
    Type      string
    Content   string
    Status    string // pending / approved / rejected
}
```

核心不是消息本身，而是“请求和响应怎么关联”。

```go
func HandleApprovalResponse(trackers map[string]*ApprovalRequest, reqID string, approve bool) {
    req := trackers[reqID]
    if req == nil {
        return
    }
    if approve {
        req.Status = "approved"
    } else {
        req.Status = "rejected"
    }
}
```

### 这节课想让你学会什么

AI agent 一旦进入多人协作，协议和状态机的重要性和传统分布式系统没有区别。

---

## 九、Autonomous Agent，用 Go 怎么理解

自治认领任务的本质是一个轮询调度器：

```go
func IdleLoop(worker string, store TaskStore, bus MessageBus) {
    ticker := time.NewTicker(5 * time.Second)
    timeout := time.After(60 * time.Second)

    for {
        select {
        case <-ticker.C:
            msgs, _ := bus.ReadInbox(worker)
            if len(msgs) > 0 {
                ResumeWork(msgs)
                return
            }

            tasks, _ := store.List()
            if task := FindUnclaimed(tasks); task != nil {
                Claim(task.ID, worker)
                ResumeWork([]TeamMessage{{Content: "auto-claimed"}})
                return
            }

        case <-timeout:
            Shutdown(worker)
            return
        }
    }
}
```

### 业务意义

这说明 agent 不再完全依赖中心化派单，而是具备“空闲时自己找事做”的能力。

---

## 十、WorktreeManager，用 Go 怎么理解

这是整个仓库里最有“系统设计味道”的部分。

你可以把它抽象成：

```go
type WorktreeManager interface {
    Create(name string, taskID int, baseRef string) (Worktree, error)
    Run(name string, command string) (string, error)
    Status(name string) (string, error)
    Remove(name string) error
}
```

而 Task 和 Worktree 的关系像这样：

```go
type TaskExecutionBinding struct {
    TaskID       int
    WorktreeName string
    Owner        string
}
```

### 这块最重要的不是 Git，而是职责分离

- `TaskManager` 管“我要干什么”
- `WorktreeManager` 管“我在哪干”

这就是控制平面和执行平面的分离。

---

## 十一、读这套代码时，建议你按这条顺序理解

### 第一步：先看不变的主循环

理解 `agent_loop` 是整个系统地基。

### 第二步：再看每节课新加了哪个管理器

比如：

- TodoManager
- SkillLoader
- TaskManager
- BackgroundManager
- MessageBus
- TeammateManager
- WorktreeManager

### 第三步：看它引入了什么新状态

每加一个管理器，通常意味着多了一种系统状态。

### 第四步：看这些状态放在哪

- 会话内
- 磁盘上
- 队列里
- 邮箱里
- worktree 索引里

这一步最关键，因为系统复杂度大多来自状态，不来自函数。

---

## 十二、你读完这一篇后应该得到什么

你不需要先学完 Python 才能看懂这个仓库。

你真正需要建立的是这套翻译关系：

- Python class -> Go struct + methods
- 文件目录 -> 状态存储层
- thread -> goroutine
- queue -> channel
- tool call -> 外部能力接口
- agent loop -> 控制循环

一旦你这样看，整个仓库就会从“Python demo”变成“一个用简洁实现讲系统设计的 AI agent 教材”。
