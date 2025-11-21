## 两种方案结合怎么样？用一个适配器，适配器里面用pydantic

这个结合方案很不错！既保持了适配器的架构清晰度，又利用了Pydantic的便利性。让我展示一下具体实现：

## 结合方案实现

### 基础实现

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Type, TypeVar, Generic, List
from datetime import datetime

T = TypeVar('T', bound=BaseModel)
E = TypeVar('E')

class BaseAdapter(Generic[E, T]):
    """基础适配器类"""
    entity_class: Type[E]
    schema_class: Type[T]

    @classmethod
    def to_schema(cls, entity: E) -> T:
        """Entity -> Schema (利用Pydantic的from_attributes)"""
        return cls.schema_class.model_validate(
            entity,
            from_attributes=True
        )

    @classmethod
    def to_entity(cls, schema: T) -> E:
        """Schema -> Entity"""
        # 获取entity需要的数据
        data = cls.prepare_entity_data(schema)
        return cls.entity_class(**data)

    @classmethod
    def prepare_entity_data(cls, schema: T) -> dict:
        """准备entity所需数据，可在子类中覆盖处理复杂映射"""
        return schema.model_dump(by_alias=False, exclude_unset=True)

    @classmethod
    def to_schema_list(cls, entities: List[E]) -> List[T]:
        """批量转换"""
        return [cls.to_schema(entity) for entity in entities]
```

### 简单场景示例

```python
# Domain Entity
class UserEntity:
    def __init__(self, id: int, name: str, email: str, created_at: datetime):
        self.id = id
        self.name = name
        self.email = email
        self.created_at = created_at

    def update_email(self, new_email: str):
        """领域逻辑"""
        self.email = new_email

# API Schema
class UserSchema(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True
    )

    id: int
    name: str
    email: str
    created_at: datetime = Field(alias="createdAt")

# 简单适配器 - 直接继承即可
class UserAdapter(BaseAdapter[UserEntity, UserSchema]):
    entity_class = UserEntity
    schema_class = UserSchema
```

### 复杂场景示例

```python
# 复杂的Domain Entity
class OrderEntity:
    def __init__(self, id: str, customer_id: int, items: List, status: str):
        self.id = id
        self.customer_id = customer_id
        self.items = items
        self.status = status
        self._total_cache = None

    def calculate_total(self) -> float:
        """计算总金额"""
        if self._total_cache is None:
            self._total_cache = sum(item.price * item.quantity for item in self.items)
        return self._total_cache

    def get_item_count(self) -> int:
        """获取商品数量"""
        return sum(item.quantity for item in self.items)

# Schema with computed fields
class OrderSchema(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True
    )

    order_id: str = Field(alias="orderId")
    customer_id: int = Field(alias="customerId")
    total_amount: float = Field(alias="totalAmount")
    item_count: int = Field(alias="itemCount")
    status: str
    items: List['OrderItemSchema']

class OrderItemSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    product_id: str
    name: str
    price: float
    quantity: int

# 复杂适配器 - 处理计算字段和嵌套转换
class OrderAdapter(BaseAdapter[OrderEntity, OrderSchema]):
    entity_class = OrderEntity
    schema_class = OrderSchema

    @classmethod
    def to_schema(cls, entity: OrderEntity) -> OrderSchema:
        """覆盖以处理计算字段"""
        # 先准备数据字典
        data = {
            'order_id': entity.id,
            'customer_id': entity.customer_id,
            'total_amount': entity.calculate_total(),  # 调用方法
            'item_count': entity.get_item_count(),      # 调用方法
            'status': entity.status,
            'items': [OrderItemAdapter.to_schema(item) for item in entity.items]
        }

        # 使用Pydantic验证和创建
        return cls.schema_class.model_validate(data)

    @classmethod
    def prepare_entity_data(cls, schema: OrderSchema) -> dict:
        """准备entity数据，处理字段映射"""
        return {
            'id': schema.order_id,
            'customer_id': schema.customer_id,
            'items': [OrderItemAdapter.to_entity(item) for item in schema.items],
            'status': schema.status
        }

class OrderItemAdapter(BaseAdapter[OrderItemEntity, OrderItemSchema]):
    entity_class = OrderItemEntity
    schema_class = OrderItemSchema
```

### 高级功能：带验证和转换的适配器

```python
class AdvancedAdapter(BaseAdapter[E, T]):
    """增强版适配器，支持自定义验证和转换"""

    @classmethod
    def to_schema(cls, entity: E, **context) -> T:
        """支持上下文的转换"""
        # 预处理
        data = cls.pre_process_to_schema(entity, **context)

        # Pydantic验证和创建
        schema = cls.schema_class.model_validate(data, from_attributes=True)

        # 后处理
        return cls.post_process_schema(schema, entity, **context)

    @classmethod
    def pre_process_to_schema(cls, entity: E, **context) -> dict:
        """转换前的预处理，子类可覆盖"""
        return entity.__dict__ if hasattr(entity, '__dict__') else entity

    @classmethod
    def post_process_schema(cls, schema: T, entity: E, **context) -> T:
        """转换后的后处理，子类可覆盖"""
        return schema

    @classmethod
    def validate_entity(cls, entity: E) -> None:
        """实体验证逻辑"""
        pass

# 使用示例
class UserProfileAdapter(AdvancedAdapter[UserEntity, UserProfileSchema]):
    entity_class = UserEntity
    schema_class = UserProfileSchema

    @classmethod
    def pre_process_to_schema(cls, entity: UserEntity, include_private=False, **context):
        """根据上下文决定包含哪些字段"""
        data = {
            'id': entity.id,
            'name': entity.name,
            'email': entity.email if include_private else None,
            'avatar_url': cls.generate_avatar_url(entity),
            'display_name': entity.display_name or entity.name
        }
        return {k: v for k, v in data.items() if v is not None}

    @classmethod
    def generate_avatar_url(cls, entity: UserEntity) -> str:
        """生成头像URL"""
        return f"https://avatar.service.com/{entity.id}"
```

### 在FastAPI中使用

```python
from fastapi import FastAPI, Depends, HTTPException
from typing import Optional

app = FastAPI()

class UserService:
    """领域服务"""
    def get_user(self, user_id: int) -> Optional[UserEntity]:
        # 获取用户逻辑
        pass

    def create_user(self, entity: UserEntity) -> UserEntity:
        # 创建用户逻辑
        pass

user_service = UserService()

# 使用依赖注入
def get_user_service() -> UserService:
    return user_service

@app.get("/users/{user_id}", response_model=UserSchema)
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service)
):
    entity = service.get_user(user_id)
    if not entity:
        raise HTTPException(status_code=404, detail="User not found")

    # 使用适配器转换
    return UserAdapter.to_schema(entity)

@app.post("/users", response_model=UserSchema)
async def create_user(
    user_data: UserSchema,
    service: UserService = Depends(get_user_service)
):
    # Schema -> Entity
    entity = UserAdapter.to_entity(user_data)

    # 业务处理
    created_entity = service.create_user(entity)

    # Entity -> Schema
    return UserAdapter.to_schema(created_entity)

@app.get("/users", response_model=List[UserSchema])
async def list_users(
    service: UserService = Depends(get_user_service)
):
    entities = service.list_users()
    # 批量转换
    return UserAdapter.to_schema_list(entities)
```

## 这种结合方案的优势

1. **架构清晰**：保持了适配器模式的清晰分层
2. **减少样板代码**：利用Pydantic的自动验证和转换
3. **灵活性高**：
    - 简单场景直接用基类
    - 复杂场景可以覆盖方法
4. **类型安全**：保持了完整的类型提示
5. **易于测试**：适配器逻辑集中，便于单元测试
6. **性能平衡**：
    - 简单字段用Pydantic自动处理
    - 复杂逻辑用自定义方法

## 最佳实践

```python
# adapters/__init__.py
from .base import BaseAdapter
from .user import UserAdapter
from .order import OrderAdapter

__all__ = ['BaseAdapter', 'UserAdapter', 'OrderAdapter']

# 可以创建工厂函数
def create_adapter(entity_type: str):
    adapters = {
        'user': UserAdapter,
        'order': OrderAdapter
    }
    return adapters.get(entity_type)
```

这种结合方案既保持了代码的整洁性，又提供了足够的灵活性来处理各种复杂场景。特别适合中大型项目使用。
