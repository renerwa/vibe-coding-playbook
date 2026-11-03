# TDD 项目实例：优惠报价 API

> 用途：通过一个最小 Web 后端项目，展示 TDD 项目的目录结构、测试代码和开发顺序。
> 最后更新：2026-11-03
> 前置知识：[TDD 开发模式](01-TDD开发模式.md)。本文不重复解释红绿重构。
> 技术栈：Python、FastAPI（Python Web API 框架）、Pydantic（请求和响应数据校验库）、pytest、FastAPI TestClient。

## 示例目标

实现一个报价接口 `POST /quotes`：

- 请求包含订单金额 `price_cents`（以“分”为单位的整数）和可选优惠码 `coupon`；
- 没有优惠码时，应付金额等于原价；
- 优惠码 `SAVE20` 表示减免 20%；
- 未知优惠码返回错误；
- 返回原价和应付金额。

使用“分”而不是小数，可以避免浮点数金额计算误差。

## 项目架构

```text
coupon-api/
├─ src/
│  └─ coupon_api/
│     ├─ __init__.py
│     ├─ main.py                 # 创建 FastAPI 应用并注册路由
│     ├─ api/
│     │  └─ quotes.py            # HTTP 接口层：接收请求、返回响应
│     ├─ schemas/
│     │  └─ quote.py             # 请求和响应结构
│     └─ services/
│        └─ pricing.py           # 优惠计算业务规则
├─ tests/
│  ├─ conftest.py                # 共享测试夹具
│  ├─ unit/
│  │  └─ test_pricing.py         # 单元测试：只测业务计算
│  └─ integration/
│     └─ test_quote_api.py       # 集成测试：真实经过 HTTP 路由
├─ pyproject.toml                # 依赖、项目元数据和 pytest 配置
└─ README.md
```

各层职责：

| 目录 | 职责 | 测试关注点 |
|---|---|---|
| `api/` | 解析请求、调用业务层、转换错误 | 状态码、响应结构 |
| `services/` | 纯业务规则 | 输入、输出和异常 |
| `schemas/` | 数据结构与基础校验 | 非法请求是否被拒绝 |
| `tests/unit/` | 快速验证业务规则 | 不启动 Web 服务 |
| `tests/integration/` | 验证多层协作 | 路由、校验、响应组合是否正确 |

这个例子没有数据库、登录和浏览器端到端测试，因为当前功能不需要这些基础设施。TDD 示例应尽量小，不要为了“像完整项目”而加入无需求的组件。

## 第一个 TDD 循环：没有优惠码

先写单元测试，此时 `pricing.py` 还不存在：

```python
# tests/unit/test_pricing.py
from coupon_api.services.pricing import calculate_payable


def test_no_coupon_keeps_original_price():
    assert calculate_payable(price_cents=9999, coupon=None) == 9999
```

运行：

```powershell
python -m pytest tests/unit/test_pricing.py -q
```

此时因为业务模块不存在而失败。这是预期的红灯，说明测试确实依赖了尚未实现的功能。

写最小实现：

```python
# src/coupon_api/services/pricing.py
def calculate_payable(price_cents: int, coupon: str | None) -> int:
    return price_cents
```

再次运行，测试通过，进入绿灯。这个实现暂时没有处理优惠码，但它只解决当前这一个行为。

## 第二个循环：支持 SAVE20

新增测试：

```python
def test_save20_applies_twenty_percent_discount():
    assert calculate_payable(price_cents=10000, coupon="SAVE20") == 8000
```

运行后出现失败，再补上最小逻辑：

```python
def calculate_payable(price_cents: int, coupon: str | None) -> int:
    if coupon == "SAVE20":
        return price_cents * 80 // 100
    return price_cents
```

测试通过。这里使用整数运算，避免金额小数误差。

## 第三个循环：拒绝未知优惠码

补充测试和异常类型：

```python
# tests/unit/test_pricing.py
import pytest

from coupon_api.services.pricing import UnknownCouponError, calculate_payable


def test_unknown_coupon_is_rejected():
    with pytest.raises(UnknownCouponError):
        calculate_payable(price_cents=10000, coupon="NOT_EXIST")
```

实现异常处理：

```python
# src/coupon_api/services/pricing.py
class UnknownCouponError(ValueError):
    pass


def calculate_payable(price_cents: int, coupon: str | None) -> int:
    if coupon is None:
        return price_cents
    if coupon == "SAVE20":
        return price_cents * 80 // 100
    raise UnknownCouponError(f"unknown coupon: {coupon}")
```

此时三个单元测试通过。接下来才进入 API 组装，而不是一开始就把所有层一起写完。

## 集成测试：验证 HTTP 契约

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient

from coupon_api.main import app


@pytest.fixture
def client():
    with TestClient(app) as test_client:
        yield test_client
```

```python
# tests/integration/test_quote_api.py
def test_quote_endpoint_returns_payable_amount(client):
    response = client.post(
        "/quotes",
        json={"price_cents": 10000, "coupon": "SAVE20"},
    )

    assert response.status_code == 200
    assert response.json() == {
        "price_cents": 10000,
        "payable_cents": 8000,
    }
```

先运行集成测试，确认接口尚不存在时得到失败结果；再实现 schema、路由和应用注册：

```python
# src/coupon_api/schemas/quote.py
from pydantic import BaseModel, Field


class QuoteRequest(BaseModel):
    price_cents: int = Field(gt=0)
    coupon: str | None = None


class QuoteResponse(BaseModel):
    price_cents: int
    payable_cents: int
```

```python
# src/coupon_api/api/quotes.py
from fastapi import APIRouter, HTTPException

from coupon_api.schemas.quote import QuoteRequest, QuoteResponse
from coupon_api.services.pricing import UnknownCouponError, calculate_payable

router = APIRouter()


@router.post("/quotes", response_model=QuoteResponse)
def create_quote(payload: QuoteRequest) -> QuoteResponse:
    try:
        payable = calculate_payable(payload.price_cents, payload.coupon)
    except UnknownCouponError as exc:
        raise HTTPException(status_code=422, detail=str(exc)) from exc

    return QuoteResponse(
        price_cents=payload.price_cents,
        payable_cents=payable,
    )
```

```python
# src/coupon_api/main.py
from fastapi import FastAPI

from coupon_api.api.quotes import router as quotes_router

app = FastAPI()
app.include_router(quotes_router)
```

## 重构：把优惠规则集中管理

前三个循环已经让测试全部通过。现在可以在不改变行为的前提下重构：

```python
# src/coupon_api/services/pricing.py
class UnknownCouponError(ValueError):
    pass


DISCOUNT_PERCENT = {"SAVE20": 20}


def calculate_payable(price_cents: int, coupon: str | None) -> int:
    if coupon is None:
        return price_cents

    discount_percent = DISCOUNT_PERCENT.get(coupon)
    if discount_percent is None:
        raise UnknownCouponError(f"unknown coupon: {coupon}")

    return price_cents * (100 - discount_percent) // 100
```

重构后重新运行所有测试。只有仍然全部通过，才能证明没有改变已有行为。

## 运行与验证

先安装项目依赖，再运行：

```powershell
python -m pytest tests/unit/test_pricing.py -q
python -m pytest -q
```

预期结果类似：

```text
....                                                     [100%]
4 passed in 0.18s
```

验证时不要只看“4 passed”，还要确认：

- 测试确实覆盖了无优惠、正常优惠、未知优惠和 HTTP 响应；
- 失败结果确实出现过，而不是实现完成后才补测试；
- 完整测试集的命令和退出码为 0；
- 用真实请求检查 `POST /quotes`，确认响应符合接口约定。

## 从这个例子迁移到什么

进入真实项目时，可以保留同样的组织方式，再按需求增加：

- 数据库仓储层和测试数据库；
- 权限、事务和外部服务失败的集成测试；
- 少量覆盖关键用户流程的端到端测试；
- CI 中的完整测试命令和报告。

不要一次性搭建全部层级。先让当前最小功能经历一次完整的红灯、绿灯和重构，再逐步扩展。

## 相关文档

- TDD 定义与循环：[TDD 开发模式](01-TDD开发模式.md)
- 与 AI 协作的步骤和提示词：[Vibe Coding 中的 TDD 工作流](02-Vibe-Coding中的TDD工作流.md)
- 测试运行器的执行和输出：[测试运行器原理与输出](03-测试运行器原理与输出.md)
