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

Alternatively you can also output your migrations as Javascript files using the `o` (alias for `--outputJs`) flag. This is useful for Javascript only projects in which TypeScript additional packages are not installed. This command, will generate a new migration file `{TIMESTAMP}-PostRefactoring.js` with the following content:

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

See, you don't need to write the queries on your own.
The rule of thumb for generating migrations is that you generate them after **each** change you made to your models. To apply multi-line formatting to your generated migration queries, use the `p` (alias for `--pretty`) flag.

## Connection option
If you need to run/revert your migrations for another connection rather than the default, use the `-c` (alias for `--connection`) and pass the config name as an argument
```
typeorm -c <your-config-name> migration:{run|revert}
```

## Using migration API to write migrations

In order to use an API to change a database schema you can use `QueryRunner`.

Example:

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

Returns all available database names including system databases.

---

```ts
getSchemas(database?: string): Promise<string[]>
```

- `database` - If database parameter specified, returns schemas of that database

Returns all available schema names including system schemas. Useful for SQLServer and Postgres only.

---

```ts
getTable(tableName: string): Promise<Table|undefined>
```

- `tableName` - name of a table to be loaded

Loads a table by a given name from the database.

---

```ts
getTables(tableNames: string[]): Promise<Table[]>
```

- `tableNames` - name of a tables to be loaded

Loads a tables by a given names from the database.

---

```ts
hasDatabase(database: string): Promise<boolean>
```

- `database` - name of a database to be checked

Checks if database with the given name exist.

---

```ts
hasSchema(schema: string): Promise<boolean>
```

- `schema` - name of a schema to be checked

Checks if schema with the given name exist. Used only for SqlServer and Postgres.

---

```ts
hasTable(table: Table|string): Promise<boolean>
```

- `table` - Table object or name

Checks if table exist.

---

```ts
hasColumn(table: Table|string, columnName: string): Promise<boolean>
```

- `table` - Table object or name
- `columnName` - name of a column to be checked

Checks if column exist in the table.

---

```ts
createDatabase(database: string, ifNotExist?: boolean): Promise<void>
```

- `database` - database name
- `ifNotExist` - skips creation if `true`, otherwise throws error if database already exist

Creates a new database.

---

```ts
dropDatabase(database: string, ifExist?: boolean): Promise<void>
```

- `database` - database name
- `ifExist` - skips deletion if `true`, otherwise throws error if database was not found

Drops database.

---

```ts
createSchema(schemaPath: string, ifNotExist?: boolean): Promise<void>
```

- `schemaPath` - schema name. For SqlServer can accept schema path (e.g. 'dbName.schemaName') as parameter.
If schema path passed, it will create schema in specified database
- `ifNotExist` - skips creation if `true`, otherwise throws error if schema already exist

Creates a new table schema.

---

```ts
dropSchema(schemaPath: string, ifExist?: boolean, isCascade?: boolean): Promise<void>
```

- `schemaPath` - schema name. For SqlServer can accept schema path (e.g. 'dbName.schemaName') as parameter.
If schema path passed, it will drop schema in specified database
- `ifExist` - skips deletion if `true`, otherwise throws error if schema was not found
- `isCascade` - If `true`, automatically drop objects (tables, functions, etc.) that are contained in the schema.
Used only in Postgres.

Drops a table schema.

---

```ts
createTable(table: Table, ifNotExist?: boolean, createForeignKeys?: boolean, createIndices?: boolean): Promise<void>
```

- `table` - Table object.
- `ifNotExist` - skips creation if `true`, otherwise throws error if table already exist. Default `false`
- `createForeignKeys` - indicates whether foreign keys will be created on table creation. Default `true`
- `createIndices` - indicates whether indices will be created on table creation. Default `true`

Creates a new table.

---

```ts
dropTable(table: Table|string, ifExist?: boolean, dropForeignKeys?: boolean, dropIndices?: boolean): Promise<void>
```

- `table` - Table object or table name to be dropped
- `ifExist` - skips dropping if `true`, otherwise throws error if table does not exist
- `dropForeignKeys` - indicates whether foreign keys will be dropped on table deletion. Default `true`
- `dropIndices` - indicates whether indices will be dropped on table deletion. Default `true`

Drops a table.

---

```ts
renameTable(oldTableOrName: Table|string, newTableName: string): Promise<void>
```

- `oldTableOrName` - old Table object or name to be renamed
- `newTableName` - new table name

Renames a table.

---

```ts
addColumn(table: Table|string, column: TableColumn): Promise<void>
```

- `table` - Table object or name
- `column` - new column

Adds a new column.

---

```ts
addColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object or name
- `columns` - new columns

Adds a new column.

---

```ts
renameColumn(table: Table|string, oldColumnOrName: TableColumn|string, newColumnOrName: TableColumn|string): Promise<void>
```

- `table` - Table object or name
- `oldColumnOrName` - old column. Accepts TableColumn object or column name
- `newColumnOrName` - new column. Accepts TableColumn object or column name

Renames a column.

---

```ts
changeColumn(table: Table|string, oldColumn: TableColumn|string, newColumn: TableColumn): Promise<void>
```

- `table` - Table object or name
- `oldColumn` -  old column. Accepts TableColumn object or column name
- `newColumn` -  new column. Accepts TableColumn object

Changes a column in the table.

---

```ts
changeColumns(table: Table|string, changedColumns: { oldColumn: TableColumn, newColumn: TableColumn }[]): Promise<void>
```

- `table` - Table object or name
- `changedColumns` - array of changed columns.
  + `oldColumn` - old TableColumn object
  + `newColumn` - new TableColumn object

Changes a columns in the table.

---

```ts
dropColumn(table: Table|string, column: TableColumn|string): Promise<void>
```

- `table` - Table object or name
- `column` - TableColumn object or column name to be dropped

Drops a column in the table.

---

```ts
dropColumns(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object or name
- `columns` - array of TableColumn objects to be dropped

Drops a columns in the table.

---

```ts
createPrimaryKey(table: Table|string, columnNames: string[]): Promise<void>
```

- `table` - Table object or name
- `columnNames` - array of column names which will be primary

Creates a new primary key.

---

```ts
updatePrimaryKeys(table: Table|string, columns: TableColumn[]): Promise<void>
```

- `table` - Table object or name
- `columns` - array of TableColumn objects which will be updated

Updates composite primary keys.

---

```ts
dropPrimaryKey(table: Table|string): Promise<void>
```

- `table` - Table object or name

Drops a primary key.

---

```ts
createUniqueConstraint(table: Table|string, uniqueConstraint: TableUnique): Promise<void>
```

- `table` - Table object or name
- `uniqueConstraint` - TableUnique object to be created

Creates new unique constraint.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `createIndex()` method instead.

---

```ts
createUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

- `table` - Table object or name
- `uniqueConstraints` - array of TableUnique objects to be created

Creates new unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `createIndices()` method instead.

---

```ts
dropUniqueConstraint(table: Table|string, uniqueOrName: TableUnique|string): Promise<void>
```

- `table` - Table object or name
- `uniqueOrName` - TableUnique object or unique constraint name to be dropped

Drops an unique constraint.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndex()` method instead.

---

```ts
dropUniqueConstraints(table: Table|string, uniqueConstraints: TableUnique[]): Promise<void>
```

- `table` - Table object or name
- `uniqueConstraints` - array of TableUnique objects to be dropped

Drops an unique constraints.

> Note: does not work for MySQL, because MySQL stores unique constraints as unique indices. Use `dropIndices()` method instead.

---

```ts
createCheckConstraint(table: Table|string, checkConstraint: TableCheck): Promise<void>
```

- `table` - Table object or name
- `checkConstraint` - TableCheck object

Creates new check constraint.

> Note: MySQL does not support check constraints.

---

```ts
createCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

- `table` - Table object or name
- `checkConstraints` - array of TableCheck objects

Creates new check constraint.

> Note: MySQL does not support check constraints.

---

```ts
dropCheckConstraint(table: Table|string, checkOrName: TableCheck|string): Promise<void>
```

- `table` - Table object or name
- `checkOrName` - TableCheck object or check constraint name

Drops check constraint.

> Note: MySQL does not support check constraints.

---

```ts
dropCheckConstraints(table: Table|string, checkConstraints: TableCheck[]): Promise<void>
```

- `table` - Table object or name
- `checkConstraints` - array of TableCheck objects

Drops check constraints.

> Note: MySQL does not support check constraints.

---

```ts
createForeignKey(table: Table|string, foreignKey: TableForeignKey): Promise<void>
```

- `table` - Table object or name
- `foreignKey` - TableForeignKey object

Creates a new foreign key.

---

```ts
createForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

- `table` - Table object or name
- `foreignKeys` - array of TableForeignKey objects

Creates a new foreign keys.

---

```ts
dropForeignKey(table: Table|string, foreignKeyOrName: TableForeignKey|string): Promise<void>
```

- `table` - Table object or name
- `foreignKeyOrName` - TableForeignKey object or foreign key name

Drops a foreign key.

---

```ts
dropForeignKeys(table: Table|string, foreignKeys: TableForeignKey[]): Promise<void>
```

- `table` - Table object or name
- `foreignKeys` - array of TableForeignKey objects

Drops a foreign keys.

---

```ts
createIndex(table: Table|string, index: TableIndex): Promise<void>
```

- `table` - Table object or name
- `index` - TableIndex object

Creates a new index.

---

```ts
createIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

- `table` - Table object or name
- `indices` - array of TableIndex objects

Creates a new indices.

---

```ts
dropIndex(table: Table|string, index: TableIndex|string): Promise<void>
```

- `table` - Table object or name
- `index` - TableIndex object or index name

Drops an index.

---

```ts
dropIndices(table: Table|string, indices: TableIndex[]): Promise<void>
```

- `table` - Table object or name
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

