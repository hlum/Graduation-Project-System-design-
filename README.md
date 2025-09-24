# Graduation-Project-System-design-


## 課題作成のフロー
```mermaid

sequenceDiagram
    participant T as 先生(Web)
    participant A as API Server
    participant DB as データベース
    participant S as 学生 (App)
    
    T->>A: 課題を出す
    A->>DB: 課題を保存する
    A->>S: 新しい課題のことを知らせる
    
```

---

## 課題提出とQueueのフロー
 ```mermaid
 sequenceDiagram
    participant S as 学生 (App)
    participant A as API Server
    participant DB as データベース
    
    S->>A: projectを載せる (github_file_link, user_id)
    A->>DB: projectを保存
    A->>DB: Job作成する (status = "queued", retry_count = 0)
    A->>DB: 他の Jobが実行中かをチェック
    
    alt 他のJobが実行中
        A->>DB: status = "queued"
    else 実行中のJobなし
        Note over A: 質問生成開始
    end
 ```

---

## 質問生成（Processing)フロー

```mermaid
sequenceDiagram
    participant A as API Server
    participant DB as データベース
    participant O as Ollama M4 MAC Server
    participant N as 通知 (FCM)
    participant S as 学生
    
    A->>DB: Job更新 (status = "processing")
    loop retry_count < 5
        A->>O: prompt/requestをOllama Serverに送信
        alt Ollama成功
            O-->>A: 生成された質問を返す（JSON）
            A->>DB: 結果をresults tableに保存
            A->>DB: Job更新 (status = "done", result_id)
            A->>N: ユーザーに質問作成完了した通知を送る
        else Ollamaエラー
            A->>DB: retry_count++
            Note over A: 少し待機してから再試行
        end
    end
    alt retry_count >= 5
        A->>DB: Job更新 (status = "failed", error_message)
        A->>N: ユーザーに失敗通知を送る 
        Note over A: 質問回答の処理開始
    end
```


---
## 質問回答の処理
```mermaid
sequenceDiagram
    participant S as 学生 (App)
    participant A as API Server
    participant DB as データベース
    
    S->>A: 質問に答える
    A->>DB: answers tableに保存
```
---

## 全体フロー
```mermaid
	sequenceDiagram
    participant S as 学生 (App)
    participant T as 先生(Web)
    participant A as API Server
    participant DB as データベース
    participant O as Ollama M4 MAC Server
    participant N as 通知 (FCM)
    
	T->>A: 課題を出す
	A->>DB: 課題を保存する
	A->>S: 新しい課題のことを知らせる
	
    S->>A: projectを載せる (github_file_link, user_id)
    A->>DB: projectを保存
    A->>DB: Job作成する (status = "queued", retry_count = 0)
    A->>DB: 他の Jobが実行中かをチェック
    
    alt 他のJobが実行中
        A->>DB: status = "queued"
    else 実行中のJobなし
        A->>DB: Job更新 (status = "processing")
        loop retry_count < 5
            A->>O: prompt/requestをOllama Serverに送信
            alt Ollama成功
                O-->>A: 生成された質問を返す（JSON）
                A->>DB: 結果をresults tableに保存
                A->>DB: Job更新 (status = "done", result_id)
                A->>N: ユーザーに質問作成完了した通知を送る
            else Ollamaエラー
                A->>DB: retry_count++
                Note over A: 少し待機してから再試行
            end
        end
        alt retry_count >= 5
            A->>DB: Job更新 (status = "failed", error_message)
            A->>N: ユーザーに失敗通知を送る（再試行ボタン付き）
        end
    end
    
    loop status="queued"のjobがある
        A->>DB: 次のjobを取得 (status="queued")
        A->>DB: Job更新 (status="processing")
        loop retry_count < 5
            A->>O: prompt/requestをOllama Serverに送信
            alt Ollama成功
                O-->>A: 生成された質問を返す（JSON）
                A->>DB: 結果をresults tableに保存
                A->>DB: Job更新 (status = "done", result_id)
                A->>N: ユーザーに質問作成完了した通知を送る
            else Ollamaエラー
                A->>DB: retry_count++
                Note over A: 少し待機してから再試行
            end
        end
        alt retry_count >= 5
            A->>DB: Job更新 (status = "failed", error_message)
            A->>N: ユーザーに失敗通知を送る（再試行ボタン付き）
        end
    end
    
    S->>A: 質問に答える
    A->>DB: answers tableに保存
    
    Note over S,N: ユーザーが再試行を選択した場合
    S->>A: 失敗したjobを再試行
    A->>DB: Job更新 (status = "queued", retry_count = 0)
    Note over A: 上記のループが再開される

```



[[Database]]
[[API実装]]
[[Cloudfare tunnel setting up]]
[[Ollama]]
