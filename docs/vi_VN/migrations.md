# Migrations

* [Cách migrations hoạt động](#how-migrations-work)
* [Tạo một migration mới](#creating-a-new-migration)
* [Chạy và revert migrations](#running-and-reverting-migrations)
* [Generating migrations](#generating-migrations)
* [Connection option](#connection-option)
* [Sử dụng migration API để viết migrations](#using-migration-api-to-write-migrations)

## Cách migrations hoạt động

Khi đã đưa lên môi trường production, bạn sẽ cần đưa những thay đổi của model vào trong database.
Về cơ bản thì việc sử dụng `synchronize: true` trên môi trường production sau khi đã đưa data vào database cho schema synchronization là khá nguy hiểm. Đây là lúc migrations sẽ phát huy tác dụng của nó.

Migration là một file chứa các câu sql queries để update database schema và áp dụng những sự thay đổi đó cho database hiện thời.

Giả sử bạn đang có một database và một post entity:

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Post {

    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    title: string;

    @Column()
    text: string;

}
```

Và entity của bạn làm việc ở môi trường production khoảng hơn một tháng mà không có bất kì sự thay đổi nào về schema. Hiện tại bạn đang có cả ngàn bài posts bên trong database.

Và giờ bạn cần tạo ra một sự thay đổi, cụ thể là thay đổi `title` thành `name`. Bạn sẽ làm như thế nào?

Bạn cần tạo một migration mới với câu sql như sau (áp dụng cho postgres):

```sql
ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name";
```

Sau khi bạn chạy câu sql query này, database schema của bạn đã được thay đổi.

TypeORM cung cấp cho bạn một nơi viết các câu sql queries và chạy chúng khi cần thiết.
Nới đó được gọi là "migrations".

## Tạo một migration mới

**Yêu cầu**: [Cài đặt CLI](./using-cli.md#installing-cli)

Trước khi tạo một migration mới bạn cần phải thiết lập connection options:

```json
{
    "type": "mysql",
    "host": "localhost",
    "port": 3306,
    "username": "test",
    "password": "test",
    "database": "test",
    "entities": ["entity/*.js"],
    "migrationsTableName": "custom_migration_table",
    "migrations": ["migration/*.js"],
    "cli": {
        "migrationsDir": "migration"
    }
}
```

Ở đây chúng ta setup 3 options:
* `"migrationsTableName": "migrations"` - Sử dụng option này khi bạn muốn migration table name khác với `"migrations"`.
* `"migrations": ["migration/*.js"]` - chỉ ra rằng typeorm phải loads các migrations từ thư mục "migration".
* `"cli": { "migrationsDir": "migration" }` - chỉ ra rằng CLI phải tạo các migrations mới trong thưc mục "migration".

Khi đã setup connection options xong, bạn đã có thể tạo một migration mới bằng CLI như sau:

```
typeorm migration:create -n PostRefactoring
```

Ở đây, `PostRefactoring` là tên của migration - bạn có thể chỉ định bất kì tên nào mà bạn muốn.
Sau khi chạy câu lệnh trên bạn có thể thấy một file mới được tạo ra tại thư mục "migration" có tên là `{TIMESTAMP}-PostRefactoring.ts` với `{TIMESTAMP}` là thời điểm mà file migration được tạo ra.
Giờ bạn có thể mở file lên và thêm câu queries sql migration mà bạn mong muốn.

Bạn sẽ nhìn thấy nội dung như phía dưới đây ở trong migration:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {

    }

    async down(queryRunner: QueryRunner): Promise<void> {

    }


}
```

Có 2 methods bạn cần phải hoàn chỉnh đó là: `up` và `down`.
`up` sẽ chứa code bạn cần để thực thi migration.
`down` sẽ revert những gì mà `up` làm.
`down` method được sử dụng để revert migration cuối cùng.

Ở trong `up` và `down` bạn có một `QueryRunner` object.
Mọi thao tác với database đều được thực hiện thông qua object này.
Đọc thêm về [query runner](./query-runner.md).

Hãy cùng xem migration của chúng ta sẽ trông như thế nào với những sự thay đổi của `Post`:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" RENAME COLUMN "title" TO "name"`);
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" RENAME COLUMN "name" TO "title"`); // revert những gì đã được chạy trong up
    }
}
```

## Chạy và revert migrations

Khi đã có migration để chạy trên môi trường production, bạn có thể sử dụng CLI để chạy như sau:

```
typeorm migration:run
```

**`typeorm migration:create` và `typeorm migration:generate` sẽ tạo ra các files `.ts`, chỉ trừ khi bạn sử dụng `o` flag (xem thêm tại [Generating migrations](#generating-migrations)). Câu lệnh `migration:run` và `migration:revert` chỉ hoạt động với `.js` files. Do đó các typescript files cần được compile trước khi chạy các câu lệnh trên.** Ngoài ra bạn có thể sử dụng `ts-node` kết hợp với `typeorm` để chạy các `.ts` migration files.

Ví dụ với `ts-node`:
```
ts-node --transpile-only ./node_modules/typeorm/cli.js migration:run
```

Ví dụ với `ts-node` không sử dụng `node_modules` trực tiếp:
```
ts-node $(yarn bin typeorm) migration:run
```

Câu lệnh này sẽ thực thi mọi pending migrations theo thứ tự được sắp xếp bởi timestamps.
Có nghĩa rằng mọi sql queries được viết trong `up` methods sẽ được thực thi.
Và cuối cùng thì bạn cũng đã có được một database schema được cập nhật mới nhất.

Nếu bạn muốn revert những sự thay đổi vừa rồi, bạn có thể chạy:

```
typeorm migration:revert
```

Câu lệnh này sẽ thực thi `down` trong migration gần nhất được thực thi.
Nếu bạn muốn revert nhiều migrations, bạn cần phải chạy câu lệnh này nhiều lần.

## Generating migrations

TypeORM có khả năng tự động tạo ra các migration files tương ứng với schema mới nhất được cập nhật.

Giả sử bạn có `Post` entity với cột `title`, bạn đã tiến hành thay đổi nó thành `name`.
Bạn có thể chạy câu lệnh sau:

```
typeorm migration:generate -n PostRefactoring
```

Và nó sẽ tạo ra một migration mới với tên gọi `{TIMESTAMP}-PostRefactoring.ts` với nội dung như sau:

```typescript
import {MigrationInterface, QueryRunner} from "typeorm";

export class PostRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name"`);
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "name" RENAME TO "title"`);
    }


}
```

Ngoài ra bạncũng có thể chuyển đầu ra của migrations dưới dạng các Javascript files bằng cách sử dụng `o` flag (viết tắt của `--outputJs`). Flag này rất hữu dụng cho các dự án chỉ sử dụng Javascript và đồng thời các packages của Typescript cũng không được cài đặt. Câu lệnh này sẽ tạo ra một migration file mới có tên là `{TIMESTAMP}-PostRefactoring.js` với nội dung như sau:

```javascript
const { MigrationInterface, QueryRunner } = require("typeorm");

module.exports = class PostRefactoringTIMESTAMP {

    async up(queryRunner) {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name"`);
    }

    async down(queryRunner) {
        await queryRunner.query(`ALTER TABLE "post" ALTER COLUMN "title" RENAME TO "name"`);
    }
}
```

Bạn thấy đó, chúng ta không cần phải viết các câu queries khi muốn thay đổi cấu trúc của database nữa.
Quy tắt ở đây đó là bạn sẽ sinh ra các file migrations sau mỗi thay đổi trong models của bạn. Để áp dụng multi-line format cho migration queries mà bạn tạo ra, bạn có thể sử dụng `p` flag (viết tắt của `--pretty`).

## Connection option

Nếu bạn muốn chạy/ revert migrations cho các connection khác ngoài default connection, hãy sử dụng `-c` flag (viết tắt của `--connection`) và truyền config name như là tham số
```
typeorm -c <your-config-name> migration:{run|revert}
```

## Sử dụng migration API để viết migrations

Để có thể sử dụng API cho mục đích thay đổi database schema, bạn có thể sử dụng `QueryRunner`.

Ví dụ:

```ts
import {MigrationInterface, QueryRunner, Table, TableIndex, TableColumn, TableForeignKey } from "typeorm";

export class QuestionRefactoringTIMESTAMP implements MigrationInterface {

    async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.createTable(new Table({
            name: "question",
            columns: [
                {
                    name: "id",
                    type: "int",
                    isPrimary: true
                },
                {
                    name: "name",
                    type: "varchar",
                }
            ]
        }), true)

        await queryRunner.createIndex("question", new TableIndex({
            name: "IDX_QUESTION_NAME",
            columnNames: ["name"]
        }));

        await queryRunner.createTable(new Table({
            name: "answer",
            columns: [
                {
                    name: "id",
                    type: "int",
                    isPrimary: true
                },
                {
                    name: "name",
                    type: "varchar",
                },
                {
                  name: 'created_at',
                  type: 'timestamp',
                  default: 'now()'
                }
            ]
        }), true);

        await queryRunner.addColumn("answer", new TableColumn({
            name: "questionId",
            type: "int"
        }));

        await queryRunner.createForeignKey("answer", new TableForeignKey({
            columnNames: ["questionId"],
            referencedColumnNames: ["id"],
            referencedTableName: "question",
            onDelete: "CASCADE"
        }));
    }

    async down(queryRunner: QueryRunner): Promise<void> {
        const table = await queryRunner.getTable("answer");
        const foreignKey = table.foreignKeys.find(fk => fk.columnNames.indexOf("questionId") !== -1);
        await queryRunner.dropForeignKey("answer", foreignKey);
        await queryRunner.dropColumn("answer", "questionId");
        await queryRunner.dropTable("answer");
        await queryRunner.dropIndex("question", "IDX_QUESTION_NAME");
        await queryRunner.dropTable("question");
    }

}
```

---

```ts
getDatabases(): Promise<string[]>
```

Trả về toàn bộ các database names bao gồm cả các system databases.

---

```ts
getSchemas(database?: string): Promise<string[]>
```

- `database` - Nếu database paramêtr được chỉ định, hàm này sẽ trả về schemas của database đó

Trả về toàn bộ các schema names có sẵn bao gồm cả system schemas. Chỉ hữu dụng cho SQLServer và Postgres.

---

```ts
getTable(tableName: string): Promise<Table|undefined>
```

- `tableName` - tên của bảng sẽ được loaded

Load bảng với tên được chỉ định từ database.

---

```ts
getTables(tableNames: string[]): Promise<Table[]>
```

- `tableNames` - tên của các bảng sẽ được loaded

Load các bảng với tên được chỉ định từ database.

---

```ts
hasDatabase(database: string): Promise<boolean>
```

- `database` - tên của database được kiểm tra.

Kiểm tra xem database với tên được chỉ định có tồn tại hay không.

---

```ts
hasSchema(schema: string): Promise<boolean>
```

- `schema` - tên của schema sẽ được checked

Kiểm tra schema với tên được chỉ định có tồn tại hay không. Chỉ áp dụng được cho SqlServer và Postgres.

---

```ts
hasTable(table: Table|string): Promise<boolean>
```

- `table` - Table object hoặc tên bảng

Kiểm tra xem bảng có tồn tại hay không.

---

```ts
hasColumn(table: Table|string, columnName: string): Promise<boolean>
```

- `table` - Table object hoặc tên bảng
- `columnName` - tên của cột sẽ được kiểm tra

Kiểm tra xem cột có tồn tại trong bảng hay không.

---

```ts
createDatabase(database: string, ifNotExist?: boolean): Promise<void>
```

- `database` - tên của database
- `ifNotExist` - bỏ qua việc tạo database nếu bằng `true`, ngược lại sẽ đưa ra lỗi nếu database đã tồn tại

Tạo một database mới.

---

```ts
dropDatabase(database: string, ifExist?: boolean): Promise<void>
```

- `database` - tên của database
- `ifExist` - bỏ qua việc xoá database nếu bằng `true`,  ngược lại sẽ đưa ra lỗi nếu database không tồn tại

Xoá bỏ database.

---

```ts
createSchema(schemaPath: string, ifNotExist?: boolean): Promise<void>
```

- `schemaPath` - tên của schema. Với SqlServer ta có thể truyền schema path (VD: 'dbName.schemaName') như là tham số.
Nếu schema path được truyền, nó sẽ tạo ra schema trong một database cụ thể
- `ifNotExist` - bỏ qua nếu có giá trị là `true`, ngược lại sẽ đưa ra lỗi nếu schema đã tồn tại

Tạo một table schema mới.

---

```ts
dropSchema(schemaPath: string, ifExist?: boolean, isCascade?: boolean): Promise<void>
```

- `schemaPath` - tên của schema. Với SqlServer ta có thể truyền schema path (VD: 'dbName.schemaName') như là tham số.
Nếu schema path được truyền, nó sẽ xoá bỏ schema trong một database cụ thể
- `ifExist` - bỏ qua việc xoá bỏ nếu có giá trị là `true`, ngược lại sẽ đưa ra lỗi nếu schema không tồn tại
- `isCascade` - Nếu bằng `true`, sẽ tự động bỏ đi các objects (bảng, function, ...) có trong schema.
Chỉ sử dụng được với Postgres.

Xoá bỏ một table schema.

---

```ts
createTable(table: Table, ifNotExist?: boolean, createForeignKeys?: boolean, createIndices?: boolean): Promise<void>
```

- `table` - Table object.
- `ifNotExist` - bỏ qua việc tạo bảng nếu bằng `true`, ngược lại sẽ đưa ra lỗi nếu bảng đã tồn tại. Giá trị mặc định là `false`
- `createForeignKeys` - được chỉ định khi muốn tạo khoá ngoại cùng lúc với quá trình tạo bảng. Giá trị mặc định là `true`
- `createIndices` - được chỉ định khi muốn tạo các indices cùng lúc với quá trình tạo bảng. Giá trị mặc định là `true`

Tạo một bảng mới.

---

```ts
dropTable(table: Table|string, ifExist?: boolean, dropForeignKeys?: boolean, dropIndices?: boolean): Promise<void>
```

- `table` - Table object hoặc tên bảng sẽ bị xoá bỏ
- `ifExist` - bỏ qua việc tạo bảng nếu bằng `true`, ngược lại sẽ đưa ra lỗi nếu bảng không tồn tại
- `dropForeignKeys` - được chỉ định khi muốn xoá bỏ luôn các khoá ngoại khi tiến hành xoá bảng. Giá trị mặc định là `true`
- `dropIndices` - được chỉ định khi indices sẽ bị loại bỏ khi tiến hành xoá bảng. Giá trị mặc định là `true`

Xoá bỏ một bảng.

---

```ts
renameTable(oldTableOrName: Table|string, newTableName: string): Promise<void>
```

- `oldTableOrName` - tên cũ của bảng
- `newTableName` - tên mới của bảng

Thay đổi tên bảng.

---

```ts
addColumn(table: Table|string, column: TableColumn): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `column` - cột mới

Thêm cột mới.

---

```ts
addColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `columns` - các cột mới

Thêm nhiều cột mới vào bảng.

---

```ts
renameColumn(table: Table|string, oldColumnOrName: TableColumn|string, newColumnOrName: TableColumn|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `oldColumnOrName` - cột cũ. Có thể là TableColumn object hoặc tên cột
- `newColumnOrName` - cột mới. Có thể là TableColumn object hoặc tên cột

Thay đổi tên cột.

---

```ts
changeColumn(table: Table|string, oldColumn: TableColumn|string, newColumn: TableColumn): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `oldColumn` -  cột cũ. Có thể là TableColumn object hoặc tên cột
- `newColumn` -  cột mới. Chỉ chấp nhận TableColumn object

Thay đổi một cột trong bảng.

---

```ts
changeColumns(table: Table|string, changedColumns: { oldColumn: TableColumn, newColumn: TableColumn }[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `changedColumns` - mảng chứa thông tin thay đổi cột.
  + `oldColumn` - TableColumn object cũ
  + `newColumn` - TableColumn object mới

Thay đổi một hoặc nhiều cột trong bảng.

---

```ts
dropColumn(table: Table|string, column: TableColumn|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `column` - TableColumn object hoặc tên cột sẽ bị xoá bỏ

Xoá một cột trong bảng.

---

```ts
dropColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `columns` - mảng của các TableColumn objects sẽ bị xoá bỏ

Xoá bỏ một hoặc nhiều cột trong bảng.

---

```ts
createPrimaryKey(table: Table|string, columnNames: string[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `columnNames` - mảng chứa tên các cột sẽ trở thành primary

Tạo một primary key mới.

---

```ts
updatePrimaryKeys(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `columns` - mảng chứa các objects TableColumn sẽ được cập nhật

Cập nhật composite primary keys.

---

```ts
dropPrimaryKey(table: Table|string): Promise<void>
```

- `table` - Table object hoặc tên bảng

Drops a primary key.

---

```ts
createUniqueConstraint(table: Table|string, uniqueConstraint: TableUnique): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `uniqueConstraint` - TableUnique object sẽ được tạo

Tạo một ràng buộc unique mới

> Chú ý: không dùng cho MySQL, vì MySQL lưu trữ ràng buộc unique dưới dạng unique indices. Sử dụng `createIndex()` method để thay thế.

---

```ts
createUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `uniqueConstraints` - array of TableUnique objects to be created

Creates new unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `createIndices()` method instead.

---

```ts
dropUniqueConstraint(table: Table|string, uniqueOrName: TableUnique|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `uniqueOrName` - TableUnique object or unique constraint name to be dropped

Drops an unique constraint.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndex()` method instead.

---

```ts
dropUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `uniqueConstraints` - array of TableUnique objects to be dropped

Drops an unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndices()` method instead.

---

```ts
createCheckConstraint(table: Table|string, checkConstraint: TableCheck): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `checkConstraint` - TableCheck object

Creates new check constraint.

> Note: MySQL does not support check constraints.

---

```ts
createCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `checkConstraints` - array of TableCheck objects

Creates new check constraint.

> Note: MySQL does not support check constraints.

---

```ts
dropCheckConstraint(table: Table|string, checkOrName: TableCheck|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `checkOrName` - TableCheck object or check constraint name

Drops check constraint.

> Note: MySQL does not support check constraints.

---

```ts
dropCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `checkConstraints` - array of TableCheck objects

Drops check constraints.

> Note: MySQL does not support check constraints.

---

```ts
createForeignKey(table: Table|string, foreignKey: TableForeignKey): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `foreignKey` - TableForeignKey object

Creates a new foreign key.

---

```ts
createForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `foreignKeys` - array of TableForeignKey objects

Creates a new foreign keys.

---

```ts
dropForeignKey(table: Table|string, foreignKeyOrName: TableForeignKey|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `foreignKeyOrName` - TableForeignKey object or foreign key name

Drops a foreign key.

---

```ts
dropForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `foreignKeys` - array of TableForeignKey objects

Drops a foreign keys.

---

```ts
createIndex(table: Table|string, index: TableIndex): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `index` - TableIndex object

Creates a new index.

---

```ts
createIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `indices` - array of TableIndex objects

Creates a new indices.

---

```ts
dropIndex(table: Table|string, index: TableIndex|string): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `index` - TableIndex object or index name

Drops an index.

---

```ts
dropIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

- `table` - Table object hoặc tên bảng
- `indices` - array of TableIndex objects

Drops an indices.

---

```ts
clearTable(tableName: string): Promise<void>
```

- `tableName` - table name

Clears all table contents.

> Note: this operation uses SQL's TRUNCATE query which cannot be reverted in transactions.

---

```ts
enableSqlMemory(): void
```

Enables special query runner mode in which sql queries won't be executed, instead they will be memorized into a special variable inside query runner.
You can get memorized sql using `getMemorySql()` method.

---

```ts
disableSqlMemory(): void
```

Disables special query runner mode in which sql queries won't be executed. Previously memorized sql will be flushed.

---

```ts
clearSqlMemory(): void
```

Flushes all memorized sql statements.

---

```ts
getMemorySql(): SqlInMemory
```

- returns `SqlInMemory` object with array of `upQueries` and `downQueries` sql statements

Gets sql stored in the memory. Parameters in the sql are already replaced.

---

```ts
executeMemoryUpSql(): Promise<void>
```

Executes memorized up sql queries.

---

```ts
executeMemoryDownSql(): Promise<void>
```

Executes memorized down sql queries.

---

