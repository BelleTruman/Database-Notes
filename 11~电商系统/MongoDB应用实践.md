# MongoDB 在电商系统中的应用

MongoDB 作为文档型数据库，在电商系统中主要用于存储灵活模式的数据，如商品详情、用户画像、评论系统等。它的 Schema-less 特性使其非常适合需要频繁变更结构的业务场景。

## 应用场景

### 1. 商品目录

商品属性多样且不固定，使用 MongoDB 可以灵活存储各类商品的不同属性。

#### 商品集合设计

```javascript
// products 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "productId": 1001,
  "productNo": "PROD20231207001",
  "productName": "iPhone 15 Pro 256GB",
  "category": {
    "categoryId": 100,
    "categoryName": "手机数码",
    "path": "/电子产品/手机数码"
  },
  "brand": "Apple",
  "price": 7999.00,
  "originalPrice": 8999.00,
  "mainImage": "https://example.com/images/iphone15pro.jpg",
  "images": [
    "https://example.com/images/iphone15pro-1.jpg",
    "https://example.com/images/iphone15pro-2.jpg",
    "https://example.com/images/iphone15pro-3.jpg"
  ],
  "description": "全新设计的 iPhone 15 Pro",
  "detail": {
    "html": "<div>详细描述...</div>",
    "features": [
      "A17 Pro 芯片",
      "钛金属设计",
      "灵动岛"
    ]
  },
  "specifications": {
    "屏幕尺寸": "6.1英寸",
    "分辨率": "2556x1179",
    "处理器": "A17 Pro",
    "内存": "8GB",
    "存储": "256GB",
    "电池": "3274mAh",
    "颜色": ["原色钛金属", "蓝色钛金属", "白色钛金属", "黑色钛金属"]
  },
  "skus": [
    {
      "skuId": 10001,
      "skuNo": "SKU20231207001",
      "attributes": {
        "颜色": "原色钛金属",
        "容量": "256GB"
      },
      "price": 7999.00,
      "stock": 100,
      "image": "https://example.com/images/iphone15pro-natural.jpg"
    }
  ],
  "tags": ["新品", "热卖", "5G"],
  "status": 1,
  "stock": 500,
  "sales": 1200,
  "viewCount": 5000,
  "rating": {
    "average": 4.8,
    "count": 320
  },
  "createdAt": ISODate("2023-12-07T10:00:00Z"),
  "updatedAt": ISODate("2023-12-07T10:00:00Z")
}
```

#### 商品查询操作

```javascript
// 查询指定分类的商品
db.products.find({
  "category.categoryId": 100,
  "status": 1
}).sort({ sales: -1 }).limit(20)

// 按价格区间查询
db.products.find({
  price: { $gte: 3000, $lte: 8000 },
  status: 1
})

// 按标签查询
db.products.find({
  tags: { $in: ["新品", "热卖"] }
})

// 更新商品销量
db.products.updateOne(
  { productId: 1001 },
  { 
    $inc: { sales: 1, viewCount: 1 },
    $set: { updatedAt: new Date() }
  }
)
```

### 2. 用户画像

用户画像数据结构灵活，适合用 MongoDB 存储。

```javascript
// user_profiles 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439012"),
  "userId": 10001,
  "basicInfo": {
    "nickname": "张三",
    "gender": "male",
    "age": 28,
    "birthday": ISODate("1995-06-15T00:00:00Z"),
    "location": {
      "province": "广东省",
      "city": "深圳市",
      "district": "南山区"
    }
  },
  "tags": ["数码爱好者", "运动达人", "美食控"],
  "interests": [
    { "category": "手机数码", "score": 0.9 },
    { "category": "运动户外", "score": 0.7 },
    { "category": "美食生鲜", "score": 0.6 }
  ],
  "preferences": {
    "brands": ["Apple", "Nike", "Adidas"],
    "priceRange": {
      "min": 500,
      "max": 5000
    },
    "shoppingTime": ["evening", "weekend"]
  },
  "behavior": {
    "totalOrders": 45,
    "totalAmount": 35000.00,
    "avgOrderAmount": 777.78,
    "lastOrderTime": ISODate("2023-12-05T14:30:00Z"),
    "favoriteCategories": [
      { "categoryId": 100, "categoryName": "手机数码", "count": 15 },
      { "categoryId": 200, "categoryName": "运动户外", "count": 12 }
    ]
  },
  "creditScore": 850,
  "memberLevel": "gold",
  "createdAt": ISODate("2022-01-15T10:00:00Z"),
  "updatedAt": ISODate("2023-12-07T10:00:00Z")
}
```

#### 用户画像查询

```javascript
// 查找目标用户群（金牌会员 + 高消费）
db.user_profiles.find({
  memberLevel: "gold",
  "behavior.avgOrderAmount": { $gte: 500 },
  creditScore: { $gte: 800 }
})

// 统计用户兴趣分布
db.user_profiles.aggregate([
  { $unwind: "$interests" },
  { $group: {
      _id: "$interests.category",
      count: { $sum: 1 },
      avgScore: { $avg: "$interests.score" }
    }
  },
  { $sort: { count: -1 } }
])
```

### 3. 商品评论系统

评论系统需要支持多级回复、点赞等功能，MongoDB 的嵌套文档很适合这种场景。

```javascript
// reviews 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439013"),
  "reviewId": 20001,
  "productId": 1001,
  "userId": 10001,
  "userName": "张三",
  "userAvatar": "https://example.com/avatar/user10001.jpg",
  "orderId": 30001,
  "rating": 5,
  "content": "非常不错的手机，性能强劲，拍照效果好！",
  "images": [
    "https://example.com/reviews/img1.jpg",
    "https://example.com/reviews/img2.jpg"
  ],
  "attributes": {
    "颜色": "原色钛金属",
    "容量": "256GB"
  },
  "isAnonymous": false,
  "likes": 125,
  "dislikes": 3,
  "helpful": 120,
  "replies": [
    {
      "replyId": 200001,
      "userId": 10002,
      "userName": "李四",
      "content": "请问电池续航怎么样？",
      "createdAt": ISODate("2023-12-06T15:30:00Z")
    },
    {
      "replyId": 200002,
      "userId": 10001,
      "userName": "张三",
      "content": "续航很不错，正常使用一天没问题",
      "replyTo": 200001,
      "createdAt": ISODate("2023-12-06T16:00:00Z")
    }
  ],
  "merchantReply": {
    "content": "感谢您的评价！",
    "createdAt": ISODate("2023-12-06T17:00:00Z")
  },
  "status": 1,
  "isTop": false,
  "createdAt": ISODate("2023-12-05T20:00:00Z"),
  "updatedAt": ISODate("2023-12-07T10:00:00Z")
}
```

#### 评论操作

```javascript
// 查询商品评论（按点赞数排序）
db.reviews.find({
  productId: 1001,
  status: 1
}).sort({ likes: -1, createdAt: -1 }).limit(20)

// 添加回复
db.reviews.updateOne(
  { reviewId: 20001 },
  { 
    $push: {
      replies: {
        replyId: 200003,
        userId: 10003,
        userName: "王五",
        content: "我也买了，确实不错",
        createdAt: new Date()
      }
    },
    $set: { updatedAt: new Date() }
  }
)

// 点赞评论
db.reviews.updateOne(
  { reviewId: 20001 },
  { $inc: { likes: 1, helpful: 1 } }
)

// 统计商品评分分布
db.reviews.aggregate([
  { $match: { productId: 1001, status: 1 } },
  { $group: {
      _id: "$rating",
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: -1 } }
])
```

### 4. 购物车（持久化）

```javascript
// shopping_carts 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439014"),
  "userId": 10001,
  "items": [
    {
      "productId": 1001,
      "skuId": 10001,
      "productName": "iPhone 15 Pro 256GB",
      "skuName": "原色钛金属 256GB",
      "price": 7999.00,
      "quantity": 1,
      "image": "https://example.com/images/iphone15pro.jpg",
      "selected": true,
      "addedAt": ISODate("2023-12-05T10:00:00Z")
    },
    {
      "productId": 1002,
      "skuId": 10002,
      "productName": "AirPods Pro 2",
      "price": 1899.00,
      "quantity": 1,
      "image": "https://example.com/images/airpods.jpg",
      "selected": true,
      "addedAt": ISODate("2023-12-06T14:00:00Z")
    }
  ],
  "totalItems": 2,
  "totalAmount": 9898.00,
  "updatedAt": ISODate("2023-12-07T10:00:00Z")
}
```

#### 购物车操作

```javascript
// 添加商品到购物车
db.shopping_carts.updateOne(
  { userId: 10001 },
  {
    $push: {
      items: {
        productId: 1003,
        skuId: 10003,
        productName: "iPad Pro",
        price: 6799.00,
        quantity: 1,
        selected: true,
        addedAt: new Date()
      }
    },
    $inc: { totalItems: 1 },
    $set: { updatedAt: new Date() }
  },
  { upsert: true }
)

// 更新商品数量
db.shopping_carts.updateOne(
  { userId: 10001, "items.skuId": 10001 },
  {
    $set: { 
      "items.$.quantity": 2,
      updatedAt: new Date()
    }
  }
)

// 删除购物车商品
db.shopping_carts.updateOne(
  { userId: 10001 },
  {
    $pull: { items: { skuId: 10001 } },
    $inc: { totalItems: -1 },
    $set: { updatedAt: new Date() }
  }
)

// 清空购物车
db.shopping_carts.updateOne(
  { userId: 10001 },
  {
    $set: { 
      items: [],
      totalItems: 0,
      totalAmount: 0,
      updatedAt: new Date()
    }
  }
)
```

### 5. 优惠券系统

```javascript
// coupons 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439015"),
  "couponId": 40001,
  "couponNo": "COUPON20231207",
  "couponName": "满500减50优惠券",
  "type": "discount", // discount, cash, percentage
  "discount": {
    "type": "cash",
    "value": 50.00,
    "minAmount": 500.00
  },
  "applicableProducts": {
    "type": "category", // all, category, product
    "categoryIds": [100, 200]
  },
  "quantity": {
    "total": 10000,
    "used": 3500,
    "remaining": 6500
  },
  "limitPerUser": 1,
  "validPeriod": {
    "startTime": ISODate("2023-12-01T00:00:00Z"),
    "endTime": ISODate("2023-12-31T23:59:59Z")
  },
  "status": 1,
  "createdAt": ISODate("2023-12-01T00:00:00Z")
}

// user_coupons 集合（用户领取的优惠券）
{
  "_id": ObjectId("507f1f77bcf86cd799439016"),
  "userId": 10001,
  "couponId": 40001,
  "couponNo": "COUPON20231207",
  "status": "unused", // unused, used, expired
  "receivedAt": ISODate("2023-12-05T10:00:00Z"),
  "usedAt": null,
  "orderId": null,
  "validPeriod": {
    "startTime": ISODate("2023-12-05T00:00:00Z"),
    "endTime": ISODate("2023-12-31T23:59:59Z")
  }
}
```

### 6. 活动配置

```javascript
// promotions 集合
{
  "_id": ObjectId("507f1f77bcf86cd799439017"),
  "promotionId": 50001,
  "promotionName": "双十二大促",
  "type": "seckill", // seckill, group_buy, discount
  "products": [
    {
      "productId": 1001,
      "originalPrice": 7999.00,
      "promotionPrice": 6999.00,
      "stock": 100,
      "sold": 0,
      "limit": 1 // 每人限购数量
    }
  ],
  "rules": {
    "maxPerUser": 2,
    "needLogin": true,
    "allowCancel": false
  },
  "timeSlots": [
    {
      "startTime": ISODate("2023-12-12T10:00:00Z"),
      "endTime": ISODate("2023-12-12T12:00:00Z")
    },
    {
      "startTime": ISODate("2023-12-12T20:00:00Z"),
      "endTime": ISODate("2023-12-12T22:00:00Z")
    }
  ],
  "status": "pending", // pending, running, ended
  "createdAt": ISODate("2023-12-01T10:00:00Z")
}
```

## 索引设计

```javascript
// products 集合索引
db.products.createIndex({ productId: 1 }, { unique: true })
db.products.createIndex({ "category.categoryId": 1, status: 1, sales: -1 })
db.products.createIndex({ brand: 1, status: 1 })
db.products.createIndex({ tags: 1 })
db.products.createIndex({ createdAt: -1 })

// reviews 集合索引
db.reviews.createIndex({ reviewId: 1 }, { unique: true })
db.reviews.createIndex({ productId: 1, status: 1, likes: -1 })
db.reviews.createIndex({ userId: 1, createdAt: -1 })

// user_profiles 集合索引
db.user_profiles.createIndex({ userId: 1 }, { unique: true })
db.user_profiles.createIndex({ memberLevel: 1, "behavior.avgOrderAmount": -1 })
db.user_profiles.createIndex({ tags: 1 })

// shopping_carts 集合索引
db.shopping_carts.createIndex({ userId: 1 }, { unique: true })
db.shopping_carts.createIndex({ updatedAt: 1 }, { expireAfterSeconds: 2592000 }) // 30天过期
```

## 聚合查询

### 商品销售分析

```javascript
// 按分类统计销售额
db.products.aggregate([
  {
    $match: { status: 1 }
  },
  {
    $group: {
      _id: "$category.categoryName",
      totalSales: { $sum: "$sales" },
      totalRevenue: { $sum: { $multiply: ["$price", "$sales"] } },
      avgPrice: { $avg: "$price" },
      count: { $sum: 1 }
    }
  },
  {
    $sort: { totalRevenue: -1 }
  }
])

// 统计用户评价分布
db.reviews.aggregate([
  {
    $match: { productId: 1001, status: 1 }
  },
  {
    $group: {
      _id: "$rating",
      count: { $sum: 1 },
      avgLikes: { $avg: "$likes" }
    }
  },
  {
    $project: {
      rating: "$_id",
      count: 1,
      avgLikes: 1,
      percentage: {
        $multiply: [
          { $divide: ["$count", { $literal: 100 }] },
          100
        ]
      }
    }
  },
  {
    $sort: { rating: -1 }
  }
])
```

## 最佳实践

1. **合理设计文档结构**：根据查询模式设计嵌套或引用
2. **避免文档过大**：单个文档不超过 16MB
3. **使用适当的索引**：提高查询性能
4. **使用投影**：只返回需要的字段
5. **批量操作**：使用 bulkWrite 提高写入性能
6. **分片策略**：数据量大时考虑分片
7. **定期备份**：使用 mongodump 备份数据
8. **监控性能**：使用 MongoDB Compass 或 Ops Manager

## 参考资料

- [MongoDB官方文档](https://docs.mongodb.com/)
- [MongoDB实践](../../04~文档型数据库/1.Mongodb/README.md)
