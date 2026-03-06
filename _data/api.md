# API 框架

## 1. 日志系统规范 (Logging Standard)

日志库位于 `libs.logtools`，底层基于 `loguru`。

### 1.1 异常捕获与打印

* **原则**：异常必须在自己的作用域内捕获，禁止盲目使用 `try...except Exception` 包裹整个大函数。
* **强制要求**：在 `except` 块中，**必须**使用 `logger.exception`。
* ✅ **正确**：`logger.exception("入库失败")` —— 自动打印完整堆栈信息和 Trace ID。
* ❌ **错误**：`logger.error(str(e))` —— 仅打印错误文字，丢失堆栈，导致无法调试。



### 1.2 格式化要求

* **性能原则**：不要在日志消息中使用 f-string。
* ✅ **推荐**：`logger.info("任务 {} 处理完成，共 {} 条", task_name, count)`
* ❌ **不推荐**：`logger.info(f"任务 {task_name} 处理完成")`



### 1.3 日志分级与路径

| 级别 | 场景描述 |
| --- | --- |
| **DEBUG** | 原始数据打印（如 API 返回的完整 JSON）。 |
| **INFO** | 业务里程碑（任务开始、入库成功数）。 |
| **WARNING** | **可自动恢复**的异常（如：API 超时重试、空数据）。 |
| **ERROR** | **局部失败**（如：某行数据解析失败被跳过）。 |
| **CRITICAL** | **任务中断**（如：数据库连接失败、关键配置缺失）。 |

* **存储路径**：
* **默认任务**：`logs/default_task/`
* **定时任务**：通过 `ecosystem.config.js` 启动，存储于 `logs/{第一个参数名}/`（例如：`logs/test/`）。



---

## 2. 定时任务开发 (Cron Tasks)

### 2.1 任务注册

新建 API 采集任务时，必须在 `apps/` 目录下新建项目文件夹，并修改 `apps/__init__.py` 进行动态注册。

* **注册机制**：
* 使用 `if TYPE_CHECKING:` 块配置 IDE 补全。
* 使用 `_EXPORTS` 字典配置动态导入，以减少内存占用并提高启动速度。



### 2.2 调度配置

使用 PM2 管理进程，配置文件为 `ecosystem.config.js`。示例配置如下：

```javascript
{
    name: "pm2_test",           // PM2 进程名
    script: "task_launcher.py",
    args: [
        "test",                 // 参数1：项目名 (对应日志子目录)
        "cron",                 // 参数2：调度模式
        "hour=6",               // 调度周期
        "minute=10",
        "--user_id=45555",      // 通知责任人 (待实现)
        "--notify=true",        // 开启通知 (待实现)
    ],
    interpreter: VENV_PY,       // 虚拟环境 Python 路径
    kill_timeout: 86400000,
    wait_ready: true,
    max_restarts: 10,
}

```

---

## 3. 工具类与数据持久化

### 3.1 HTTP 请求 (libs/httpxtools.py)

所有 API 采集必须使用带 `retry` 机制的工具函数：

* **同步方法**：
* `request_raw`: 返回 Response 对象。
* `request`: 返回 JSON 化后的字典/列表。


* **异步方法**：
* `request_raw_async`: 返回异步 Response。
* `request_async`: 返回 JSON 化后的数据。



### 3.2 数据库 ORM (libs/sqlmodel_db.py)

项目采用基类扩展模式进行数据保存。

### 3.3 Auto build Model (tests/auto_build_model.py) OrmBaseDB里的方法

生成的model在models/下，并且需要存在的表才能够build出来

```python
from configs.db_info import db_configs
from libs.sqlmodel_db import BaseDB

if __name__ == "__main__":
    # 需要写入的表
    db_config = db_configs["finebi_dws_db"]

    db_manager = BaseDB(db_config)

    db_manager.schema = "overseas_subsidiaries"

    # 获取并生成这个两个表的 sqlmodel类型
    db_manager.make_models("dms_eu_dealer_sign_info")
```

* **导入方式**：`from libs import OrmBaseDB`

---

## 4. API 开发规范 (FastAPI)

### 4.1 目录结构

API 相关代码统一存放于 `apis/` 路径下。每个项目应拥有独立的文件夹，包含模型定义与逻辑实现。

```text
apis/
└── your_project/
    ├── __init__.py
    ├── model.py      # 数据模型
    └── router.py     # 路由定义

```

### 4.2 路由集成

1. 在项目文件夹内创建 `APIRouter`。
2. 统一导入至根目录的 `api_main.py` 中，通过 `app.include_router()` 挂载。
3. **交互式文档**：开发完成后，直接访问 `domain/docs` 进行接口调试。

