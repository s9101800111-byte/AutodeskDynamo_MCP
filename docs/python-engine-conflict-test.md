# Dynamo Python 引擎衝突對照測試（Revit 2026 / Dynamo 3.6.2）

**要回答的問題**：啟動 revit-mcp 服務，會不會讓 Dynamo 的 CPython3 節點壞掉（`TypeError: Cannot extend an incomplete type 'MulticastDelegate'`）？

## 背景（2026-10-21 下午）

| 次數 | revit-mcp 服務 | Dynamo Python | 備註 |
|---|---|---|---|
| 第 1 次（13:47 前後，專案2） | 未啟動（revit-mcp 查詢回 Error） | ✅ 正常，建出 5 道牆 | |
| 第 2 次（14:50 前後，專案1） | **已啟動**（localhost:8964） | ❌ 所有 Python 節點失敗 | 第一個節點：`Cannot extend an incomplete type 'MulticastDelegate'`；之後的節點：`__import__ returned a result with an error set` |
| 第 3 次（15:00 起，專案1） | 未啟動 | ✅ 正常 | 見下方基準 |

## 第 3 次基準：服務**未啟動**

- Revit pid 19580，啟動時間 2026-10-21 14:59:57
- Dynamo session `d42b4da0-68ec-4614-99d8-d122f287b718`
- autotest：Python 節點 `OUT = IN[0]` → Watch = **42** ✅（`get_node_values` 讀回）
- 建牆：樓層 FL1（Id 311）、牆類型 Id 85262、牆 Id 257330–257334 ✅
- Clockwork 兩個節點 Error「未載入自訂節點定義」：沒裝該套件，與本測試無關

### Revit 程序內已載入的 Python 相關 DLL（服務未啟動）

| 模組 | 版本 | 來源 |
|---|---|---|
| `Python.Runtime.dll` | 2.5.2.16017 | Dynamo（`Revit 2026\AddIns\DynamoForRevit\`） |
| `DSCPython.dll` | 3.6.2.11575 | Dynamo |
| `Python.Included.dll` | 2.5.2.12086 | Dynamo |
| `python39.DLL` | 3.9.12 | `%LOCALAPPDATA%\python-3.9.12-embed-amd64\`（Dynamo CPython3 用） |
| `pyRevitLabs.PythonNet.dll` | **3.0.3.0** | pyRevit（`%APPDATA%\pyRevit-Master\bin\netcore\`） |
| `pyRevitLabs.IronPython*.dll` | 2.7.12 | pyRevit |

**觀察**：pyRevit 的 PythonNet 3.0.3 與 Dynamo 的 Python.Runtime 2.5.2 已經**同時載入**，Dynamo Python 仍正常 → 單純「pyRevit 也在」不足以造成故障。

## 第 4 次：同一個 Revit 程序，**之後**才啟動 revit-mcp 服務

- 使用者按「MCP 服務（開）」，8964 埠開始監聽（HTTP.sys 代管，顯示 PID 4）
- Revit 程序的 Python 相關 DLL：**與第 3 次相同，沒有新增**
- 重跑與第 2 次失敗時相同的 Python 程式碼（`clr.ImportExtensions(Revit.Elements)` + `ToDSType(True)`）
  → ✅ **正常**，讀回 FL1（Id 311）、牆類型 Id 85262，無警告

**結論**：「啟動 revit-mcp 服務就會讓 Python 壞掉」**不成立**。

## 剩下的假設：啟動順序

| | 先後順序 | Python |
|---|---|---|
| 第 2 次 | revit-mcp 服務**先**啟動 → Dynamo 才第一次初始化 CPython | ❌ |
| 第 4 次 | Dynamo CPython **先**初始化（autotest）→ 之後才啟動服務 | ✅ |

## 第 5 次：重開 Revit，**先**開服務、**後**開 Dynamo

- Revit pid 47504，啟動時間 2026-10-21 15:05:58
- 8964 埠 LISTENING，且已有 MCP 客戶端連上（ESTABLISHED）
- 放第一個 Python 節點前，`python39.dll` 尚未載入（CPython 未初始化）
- 第一個 Python 節點用與第 2 次相同的程式碼 → ✅ **正常**，讀回 FL1（Id 311）、牆類型 Id 85262
- 跑完後載入的 Python DLL 與第 3 次相同（Dynamo 的 Python.Runtime 2.5.2 + python39 3.9.12，pyRevit 的 PythonNet 3.0.3），沒有其他

**結論**：啟動順序也不是原因。

## 總結（2026-10-21）

| 假設 | 結果 |
|---|---|
| 裝了 pyRevit 就會壞 | ❌ 不成立（第 3–5 次 pyRevit PythonNet 都在，Python 正常） |
| 啟動 revit-mcp 服務就會壞 | ❌ 不成立（第 4 次） |
| 服務先於 Dynamo 啟動就會壞 | ❌ 不成立（第 5 次） |

第 2 次的失敗**無法重現，先記為偶發**。第 2 次當時沒有留下 DLL 清單，是這次查不出原因的主因。

### 下次再出現 `Cannot extend an incomplete type 'MulticastDelegate'` 時

先**不要關 Revit**，當場留證據：

1. `get_node_values` 讀出第一個失敗的 Python 節點的完整錯誤訊息
2. 列出 Revit 已載入的 Python 相關 DLL，與上表比對有沒有多出的 `python3xx.dll` 或其他 Python.Runtime：
   ```powershell
   (Get-Process Revit).Modules | Where-Object { $_.ModuleName -match '^python\d+\.dll$|Python\.Runtime|PythonNet|DSCPython' } | ForEach-Object { "{0} | {1} | {2}" -f $_.ModuleName, $_.FileVersionInfo.FileVersion, $_.FileName }
   ```
3. 回想這次開 Revit 後、開 Dynamo 前，有沒有按過 pyRevit 按鈕（特別是 CPython 引擎的工具）

**解法**：關閉 Revit 重開即可恢復；壞掉的狀態會持續到 Revit 關閉為止，同一程序內所有 Python 節點都會失敗。
