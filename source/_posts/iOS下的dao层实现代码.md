---
title: iOS下的dao层实现代码
date: 2017-01-18 17:28:33
tags:
categories: iOS
---

## iOS下的dao层实现代码

#### 什么是dao层
DAO (Data Access Object) 数据访问对象是一个面向对象的接口. 直接操作数据库, 针对数据的增添,删除,修改,查找,具体为业务逻辑层或表示层提供数据服务.

<!-- more -->
#### 什么是dao层
DAO (Data Access Object) 数据访问对象是一个面向对象的接口. 直接操作数据库, 针对数据的增添,删除,修改,查找,具体为业务逻辑层或表示层提供数据服务.

#### 一个dao的例子
```
- (void)saveRemind:(RemindEntity *)remind; // 保存一个提醒
- (void)deleteRemindWithWhere:(NSString *)where; // 删除一个提醒
- (RemindEntity *) loadRemindWithRemindID:(NSString *)rid; // 读取一个提醒
- (void)deleteRemindWithRemindID:(NSString *)rid; // 删除一个提醒
- (void)deleteRemindWithWeihuaId:(NSString *)weihuaId rcdId:(int)rcdId; // 删除一个联系人或者微话好友的提醒
- (NSArray *)searchRemindWithWhere:(NSString *)where
                            order:(NSString *)order
                            limit:(NSString *)limit; // 查找提醒
- (NSInteger)remindCountWithWhere:(NSString *)where; // 返回提醒数量
- (NSArray *)allArrivalReminds; // 返回所有未过期, 下次开始时间小于现在时间的提醒
- (NSArray *)allNonOverdueReminds; // 返回所有未过期的提醒
+ (void) setupDB; // 初始化数据库
```
内部实现是用fmdb,得写大量的sql.如果我们有多个实体类, 就得写多个dao.


#### 范化的dao

下面我们来看一个范化的dao, 就不用为每个实体专门写sql了,
具体使用的时候可以继承这个类或者写个工厂类.

```
@protocol XYBaseDaoEntityProtocol <NSObject>

@required
// 返回表名
+ (NSString *)getTableName;

// 返回主键
+ (NSString *)getPrimaryKey;

@end


// 范化的本地dao类
@interface XYBaseDao : NSObject

@property (nonatomic, weak, readonly) Class entityClass;

+ (instancetype)daoWithEntityClass:(Class)aClass;

- (NSError *)saveEntity:(id)entity;
- (NSError *)saveEntityWithArray:(NSArray *)array;

- (id)loadEntityWithKey:(NSString *)key;
- (NSArray *)loadEntityWithWhere:(NSString *)where;

- (NSInteger)countWithWhere:(NSString *)where;

- (NSError *)deleteEntityWithKey:(NSString *)key;
- (NSError *)deleteEntityWithWhere:(NSString *)where;

- (void)deleteAllEntity;

@end
```
这个dao是基于LKDBHelper的, 把实体对象的类当初始化参数传进去.

```
+ (instancetype)daoWithEntityClass:(Class)aClass
{
    XYBaseDao *dao = [[[self class] alloc] initWithEntityClass:aClass];

    return dao;
}

- (instancetype)initWithEntityClass:(Class)aClass
{
    self = [super init];
    if (self)
    {
        _entityClass = aClass;
        _globalHelper = [LKDBHelper getUsingLKDBHelper];
        // 创建表
        [_globalHelper createTableWithModelClass:[_entityClass class]];
    }
    return self;
}
```


```
- (NSError *)saveEntity:(id)entity
{
    if (entity == nil)
        return [NSError errorWithDomain:[NSString stringWithFormat:@"[save error] :%@", entity] code:XYBaseDao_error_code userInfo:nil];

    LKDBHelper *globalHelper = [LKDBHelper getUsingLKDBHelper];

    if (![globalHelper insertToDB:entity])
    {
        return [NSError errorWithDomain:[NSString stringWithFormat:@"[save error] :%@", entity] code:XYBaseDao_error_code userInfo:nil];
    }

    return nil;
}
```

```
- (NSError *)saveEntityWithArray:(NSArray *)array
{
    if (array == nil || array.count == 0)
        return [NSError errorWithDomain:[NSString stringWithFormat:@"[save error] :%@", array] code:XYBaseDao_error_code userInfo:nil];

    NSMutableString *str = [NSMutableString stringWithFormat:@"[save error] :"];
    NSInteger length = str.length;

    __weak __typeof(XYBaseDao *) weakSelf = self;

    // 事务 transaction
    [_globalHelper executeDB:^(FMDatabase *db) {
        [db beginTransaction];
        for (NSObject *entity in array)
        {
            if (![weakSelf.globalHelper insertToDB:entity])
            {
                [str stringByAppendingFormat:@"%@ ", entity];
            }
        }
        [db commit];
    }];

    if (str.length > length)
    {
        return [NSError errorWithDomain:str code:XYBaseDao_error_code userInfo:nil];
    }

    return nil;
}

```

```
- (id)loadEntityWithKey:(NSString *)key
{
    if (key.length == 0)
        return nil;

    id object = [[_entityClass alloc] init];

    NSString *where = nil;

    if ([key isKindOfClass:[NSString class]])
    {
        where = [NSString stringWithFormat:@"%@ = '%@'", [_entityClass getPrimaryKey], key];
    }
    else if ([key isKindOfClass:[NSNumber class]])
    {
        where = [NSString stringWithFormat:@"%@ = %@", [_entityClass getPrimaryKey], key];
    }
    else if (1)
    {
#pragma mark- todo  多主键种类判断
        where = @"";
    }

    object = [_entityClass searchSingleWithWhere:where orderBy:nil];

    return object;
}
```

```
- (NSArray *)loadEntityWithWhere:(NSString *)where
{
    if (where.length == 0)
        return nil;

    NSArray *array = [_entityClass searchWithWhere:where orderBy:nil offset:0 count:XYBaseDao_load_maxCount];

    return array;
}
```

```
- (NSInteger)countWithWhere:(NSString *)where
{
    return [_entityClass rowCountWithWhere:where];
}
```



```
- (NSError *)deleteEntityWithKey:(NSString *)key
{
    id object = [[_entityClass alloc] init];
    [object setValue:key forKey:[_entityClass getPrimaryKey]];

    if (![object deleteToDB])
    {
        return  [NSError errorWithDomain:[NSString stringWithFormat:@"[delete error] : %@ %@", _entityClass, key] code:XYBaseDao_error_code userInfo:nil];
    }

    return nil;
}
```

```
- (NSError *)deleteEntityWithWhere:(NSString *)where
{
    if (![_entityClass deleteWithWhere:where])
    {
        return  [NSError errorWithDomain:[NSString stringWithFormat:@"[delete error] : %@ %@", _entityClass, where] code:XYBaseDao_error_code userInfo:nil];
    }

    return nil;
}
```

```
- (void)deleteAllEntity
{
    [LKDBHelper clearTableData:[_entityClass class]];
}
```

最后是demo

```
- (void)clickTestDao:(id)sender
{
    XYBaseDao *dao = [XYBaseDao daoWithEntityClass:[CarEntity class]];
    [dao deleteAllEntity];

    CarEntity *car = [[CarEntity alloc] init];
    car.name = @"a";
    car.brand = @"科鲁兹";
    car.time = 1;

    [dao saveEntity:car];

    CarEntity *car2 = [[CarEntity alloc] init];
    car2.name = @"b";
    car2.brand = @"科鲁兹";
    car2.time = 1;

    CarEntity *car3 = [[CarEntity alloc] init];
    car3.name = @"c";
    car3.brand = @"福克斯";
    car3.time = 1;

    CarEntity *car5 = [[CarEntity alloc] init];
    car5.name = @"d";
    car5.brand = @"高尔夫";
    car5.time = 1;

    NSArray *array = @[car2, car3, car5];

    [dao saveEntityWithArray:array];

    CarEntity *car4 = [dao loadEntityWithKey:@"c"];
    NSLog(@"%@", car4);

    NSArray *array2 = [dao loadEntityWithWhere:@"brand = '科鲁兹'"];
    NSLog(@"%@", array2);

    NSInteger count = [dao countWithWhere:nil];
    NSLog(@"%d", count);

    [dao deleteEntityWithKey:@"c"];
    [dao deleteEntityWithKey:@"brand = '科鲁兹'"];
}
```
