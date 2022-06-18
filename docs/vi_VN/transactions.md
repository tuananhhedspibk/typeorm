# Transactions

- [Tạo và sử dụng transactions](#creating-and-using-transactions)
	- [Chỉ định isolation levels](#specifying-isolation-levels)
- [Transaction decorators](#transaction-decorators)
- [Sử dụng `QueryRunner` để tạo và điều khiển state của một database connection đơn](#using-queryrunner-to-create-and-control-state-of-single-database-connection)

## Tạo và sử dụng transactions

Transactions được tạo bằng cách sử dụng `Connection` hoặc `EntityManager`.
Ví dụ:

```typescript
import {getConnection} from "typeorm";

await getConnection().transaction(async transactionalEntityManager => {

});
```

hoặc

```typescript
import {getManager} from "typeorm";

await getManager().transaction(async transactionalEntityManager => {

});
```

Mọi thứ bạn muốn chạy trong transaction phải được thực thi trong một hàm callback:

```typescript
import {getManager} from "typeorm";

await getManager().transaction(async transactionalEntityManager => {
    await transactionalEntityManager.save(users);
    await transactionalEntityManager.save(photos);
    // ...
});
```

Điều cần chú ý nhất khi làm việc với một transaction đó là **luôn luôn** phải sử dụng instance được cung cấp bởi entity manager -
`transactionalEntityManager` như ở ví dụ trên.
Nếu bạn sử dụng global manager (từ `getManager` hoặc manager từ connection) bạn sẽ gặp phải ít nhiều rắc rối.
Bạn cũng không thể sử dụng các classes có sử dụng global manager hoặc connection để thực thi các câu queries của chúng.
Mọi thao tác **phải** được thực hiện thông qua transaction được cung cấp bởi entity manager.

### Chỉ định isolation levels

Việc chỉ định isolation levels cho transaction có thể được thực hiện bằng cách truyền nó như tham số đầu tiên:

```typescript
import {getManager} from "typeorm";

await getManager().transaction("SERIALIZABLE", transactionalEntityManager => {

});
```

Việc triển khai isolation levels **không** phải là điều bất khả thi trên các databases.

Các database drivers sau đây hỗ trợ các isolation levels tiêu chuẩn (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`):
* MySQL
* Postgres
* SQL Server

**SQlite** mặc định cho các transactions gắn với `SERIALIZABLE`, tuy nhiên nếu *shared cache mode* được triển khai thì một transaction có thể sử dụng `READ UNCOMMITTED` isolation level.

**Oracle** chỉ hỗ trợ hai isolation levels là `READ COMMITTED` và `SERIALIZABLE`.


## Transaction decorators

Có một vài decorators cho phép bạn có thể tổ chức các transactions của mình -
`@Transaction`, `@TransactionManager` và `@TransactionRepository`.

`@Transaction` bao toàn bộ việc thực thi vào một database transaction đơn duy nhất,
và `@TransactionManager` cung cấp một transaction entity manager phải được sử dụng để thực thi các câu queries trong transaction:

```typescript
@Transaction()
save(@TransactionManager() manager: EntityManager, user: User) {
    return manager.save(user);
}
```

với isolation level:

```typescript
@Transaction({ isolation: "SERIALIZABLE" })
save(@TransactionManager() manager: EntityManager, user: User) {
    return manager.save(user);
}
```

Bạn luôn luôn **phải** sử dụng manager được cung cấp bởi `@TransactionManager`.

Ngoài ra bạn cũng có thể inject transaction repository (có sử dụng entity manager ở phía trong), bằng cách sử dụng `@TransactionRepository`:

```typescript
@Transaction()
save(user: User, @TransactionRepository(User) userRepository: Repository<User>) {
    return userRepository.save(user);
}
```

Bạn có thể inject cả các TypeORM repositories có sẵn  như `Repository`, `TreeRepository` và `MongoRepository`
(bằng cách sử dụng `@TransactionRepository(Entity) entityRepository: Repository<Entity>`)
hoặc custom repositories (các classes kế thừa các TypeORM's repositories classes có sẵn và được gắn với `@EntityRepository`)
bằng cách sử dụng `@TransactionRepository() customRepository: CustomRepository`.

## Sử dụng `QueryRunner` để tạo và điều khiển trạng thái của một database connection đơn

`QueryRunner` cung cấp một database connection đơn.
Transactions được tổ chức bởi cách query runners.
Các transactions đơn chỉ có thể được tạo ra dựa trên một query runner đơn.
Bạn có thể tự tạo một query runnner instance và sử dụng nó để điều khiển trạng thái của transaction.
Ví dụ:

```typescript
import {getConnection} from "typeorm";

// Lấy về một connection và tạo một query runner mới
const connection = getConnection();
const queryRunner = connection.createQueryRunner();

// Tạo một database connection sử dụng query runner
await queryRunner.connect();

// bây giờ chúng ta có thể thực thi bất kì câu queries nào trên query runner, lấy ví dụ:
await queryRunner.query("SELECT * FROM users");

// chúng ta cũng có thể sử dụng entity manager gắn với connection được tạo bởi query runner:
const users = await queryRunner.manager.find(User);

// hãy cùng nhau tạo một transaction mới
await queryRunner.startTransaction();

try {

    // thực thi một vài thao tác trên transaction này:
    await queryRunner.manager.save(user1);
    await queryRunner.manager.save(user2);
    await queryRunner.manager.save(photos);

    // commit transaction:
    await queryRunner.commitTransaction();

} catch (err) {

    // nếu có lỗi, ta có thể rollback lại.
    await queryRunner.rollbackTransaction();

} finally {

    // bạn cần phải release query runner với mới được tạo:
    await queryRunner.release();
}
```

Có 3 phương thức để điều khiển transactions trong `QueryRunner`:


* `startTransaction` - bắt đầu một transaction mới bên trong một query runner instance.
* `commitTransaction` - commits toàn bộ thay đổi thông qua query runner instance.
* `rollbackTransaction` - rollback toàn bộ sử thay đổi bằng query runner instance.

Xem thêm về [Query Runner](./query-runner.md).
