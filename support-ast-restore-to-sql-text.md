---
title: 十分钟成为 Contributor 系列 | 支持 AST 还原为 SQL
author: ['赵一霖']
date: 2018-12-05
summary: 为了实现一些新特性，我们需要为 AST 实现可以还原为 SQL 文本的功能，这篇教程描述如何为 AST 节点添加该功能。首先介绍一些必需的背景知识，然后介绍实现 Restore() 函数的流程，最后会展示一个例子。
tags: ['TiDB', 'Contributor', 'SQL', '社区']
---


## **背景知识**

SQL 语句发送到 TiDB 后首先会经过 parser，从文本 parse 成为 AST（抽象语法树），AST 节点与 SQL 文本结构是一一对应的，我们通过遍历整个 AST 树就可以拼接出一个与 AST 语义相同的 SQL 文本。

对 parser 不熟悉的小伙伴们可以看 [TiDB 源码阅读系列文章（五）TiDB SQL Parser 的实现](https://www.pingcap.com/blog-cn/tidb-source-code-reading-5/)。

我们当前 AST 结构定义如下（以 `ast.CreateUserStmt` 为例）：

![create-user-stmt](media/create-user-stmt.png)

我们在 `ast.Node` 接口中添加了一个 `Restore(sb *strings.Builder) error` 函数，这个函数将当前节点对应的 SQL 文本追加至参数 `sb` 中，如果节点结构无效则返回 `error`。

```
type Node interface {
    // Restore AST to SQL text and append them to `sb`.
    // return error when the AST is invalid.
    Restore(sb *strings.Builder) error
    
    ...
}
```

以 SQL 语句 `SELECT column0 FROM table0 UNION SELECT column1 FROM table1 WHERE a = 1` 为例，如下图所示，我们通过遍历整个 AST 树，递归调用每个节点的 `Restore()` 方法，即可拼接成一个完整的 SQL 文本。

![ast-tree](media/ast-tree.png)

值得注意的是，SQL 文本与 AST 是一个多对一的关系，我们不可能从 AST 结构中还原出与原 SQL 完全一致的文本，
因此我们只要保证还原出的 SQL 文本与原 SQL **语义相同** 即可。所谓语义相同，指的是由 AST 还原出的 SQL 文本再被解析为 AST 后，两颗 AST 是相等的。

我们已经完成了接口设计和测试框架，具体的`Restore()` 函数留空。因此**只需要选择一个留空的 `Restore()` 函数实现，并添加相应的测试数据，就可以提交一个 PR 了！**

## **实现 `Restore()` 函数的整体流程**

0. 最好先看看 [Proposal](https://github.com/pingcap/tidb/tree/master/docs/design/2018-11-29-ast-to-sql-text.md)、[Issue](https://github.com/pingcap/tidb/issues/8532)

1. 找到未实现的函数

    在 [Issue-pingcap/tidb#8532](https://github.com/pingcap/tidb/issues/8532) 中找到一个没有被其他贡献者认领的任务，例如 `ast/expressions.go: BetweenExpr`。
    
    在 [pingcap/parser](https://github.com/pingcap/parser) 中找到任务对应文件 `ast/expressions.go`。
    
    在文件中找到 `BetweenExpr` 结构的 `Restore` 函数：

    ```
    // Restore implements Recoverable interface.
    func (n *BetweenExpr) Restore(sb *strings.Builder) error {
        return errors.New("Not implemented")
    }
    ```

2. 实现 `Restore()` 函数

    根据 Node 节点结构和 SQL 语法实现函数功能。
    
     > 参考 [MySQL 5.7 SQL Statement Syntax](https://dev.mysql.com/doc/refman/5.7/en/sql-syntax.html)

3. 写单元测试

    参考示例在相关文件下添加单元测试。

4. 运行 `make test`，确保所有的 test case 都能跑过。

## **示例**

这里以[实现 ColumnNameExpr 的 Restore 函数 PR](https://github.com/pingcap/parser/pull/63/files) 为例，进行详细说明：

1. 首先看 `ast/expressions.go`：

    我们要实现一个 ast.Node 结构的 Restore 函数，首先清楚该结构代表什么短语，例如 ColumnNameExpr 代表列名；
    
    观察 ColumnNameExpr 结构：
    
    ```
    // ColumnNameExpr represents a column name expression.
    type ColumnNameExpr struct {
        exprNode
    
        // Name is the referenced column name.
        // 我们发现要先实现 ColumnName 的 Restore 函数
        Name *ColumnName
    
        // Refer is the result field the column name refers to.
        // The value of Refer.Expr is used as the value of the expression.
        // 观察 parser.y (3373~3401 行) 发现 parser 过程并没有对 Refer 赋值，因此忽略这个字段
        Refer *ResultField
    }
    ```
    
    实现 ColumnName 的 Restore 函数：
    
    ```
    // Restore implements Node interface.
    // ColumnName 表示列名
    func (n *ColumnName) Restore(sb *strings.Builder) error {
        // 如果 Schema 非空则写入 Schema 名
        if n.Schema.O != "" {
            // 调用 WriteName 函数追加 Name，自动添加反引号
            WriteName(sb, n.Schema.O)
            sb.WriteString(".")
        }
        // 如果 Table 名非空则写入 Table 名
        if n.Table.O != "" {
            WriteName(sb, n.Table.O)
            sb.WriteString(".")
        }
        // 写入列名
        WriteName(sb, n.Name.O)
        return nil
    }
    ```
    
    然后我们实现 ColumnNameExpr 的 Restore 函数：
    
    ```
    // Restore implements Node interface.
    func (n *ColumnNameExpr) Restore(sb *strings.Builder) error {
        err := n.Name.Restore(sb)
        if err != nil {
            return errors.Trace(err)
        }
        return nil
    }
    ```

2. 接下来给函数实现添加单元测试，参见 [pingcap/parser#75](https://github.com/pingcap/parser/pull/75), `ast/expressions_test.go`：

    ```
    // 添加测试函数
    func (tc *testExpressionsSuite) TestColumnNameExprRestore(c *C) {
        // 测试用例
        testCases := []NodeRestoreTestCase{
            {"abc", "`abc`"},
            {"`abc`", "`abc`"},
            {"`ab``c`", "`ab``c`"},
            {"sabc.tABC", "`sabc`.`tABC`"},
            {"dabc.sabc.tabc", "`dabc`.`sabc`.`tabc`"},
            {"dabc.`sabc`.tabc", "`dabc`.`sabc`.`tabc`"},
            {"`dABC`.`sabc`.tabc", "`dABC`.`sabc`.`tabc`"},
        }
        // 为了不依赖父节点实现，通过 extractNodeFunc 抽取待测节点
        extractNodeFunc := func(node Node) Node {
            return node.(*SelectStmt).Fields.Fields[0].Expr
        }
        // Run Test
        RunNodeRestoreTest(c, testCases, "select %s", extractNodeFunc)
    }
    ```
    **至此 `ColumnNameExpr` 的 `Restore` 函数实现完成，可以提交 PR 了。为了更好的理解测试逻辑，下面我们看 `RunNodeRestoreTest`**
    
    ```
    // 下面是测试逻辑，已经实现好了，不需要贡献者实现
    func RunNodeRestoreTest(c *C, nodeTestCases []NodeRestoreTestCase, template string, extractNodeFunc func(node Node) Node) {
        parser := parser.New()
        for _, testCase := range nodeTestCases {
            // 通过 template 将测试用例拼接为完整的 SQL
            sourceSQL := fmt.Sprintf(template, testCase.sourceSQL)
            expectSQL := fmt.Sprintf(template, testCase.expectSQL)
            stmt, err := parser.ParseOneStmt(sourceSQL, "", "")
            comment := Commentf("source %#v", testCase)
            c.Assert(err, IsNil, comment)
            var sb strings.Builder
            // 抽取指定节点并调用其 Restore 函数
            err = extractNodeFunc(stmt).Restore(&sb)
            c.Assert(err, IsNil, comment)
            // 通过 template 将 restore 结果拼接为完整的 SQL
            restoreSql := fmt.Sprintf(template, sb.String())
            comment = Commentf("source %#v; restore %v", testCase, restoreSql)
            // 测试 restore 结果与预期一致
            c.Assert(restoreSql, Equals, expectSQL, comment)
            stmt2, err := parser.ParseOneStmt(restoreSql, "", "")
            c.Assert(err, IsNil, comment)
            CleanNodeText(stmt)
            CleanNodeText(stmt2)
            // 测试解析的 stmt 与原 stmt 一致
            c.Assert(stmt2, DeepEquals, stmt, comment)
        }
    }
    ```
    
**不过对于 `ast.StmtNode`（例如：`ast.SelectStmt`）测试方法有些不一样，
由于这类节点可以还原为一个完整的 SQL，因此直接在 `parser_test.go` 中测试。**

下面以[实现 UseStmt 的 Restore 函数 PR](https://github.com/pingcap/parser/pull/62/files) 为例，对测试进行说明：

1. Restore 函数实现过程略。

2. 给函数实现添加单元测试，参见 `parser_test.go`：
    
    在这个示例中，只添加了几行测试数据就完成了测试：
    
    ```
    // 添加 testCase 结构的测试数据
    {"use `select`", true, "USE `select`"},
    {"use `sel``ect`", true, "USE `sel``ect`"},
    {"use select", false, "USE `select`"},
    ```
    
    我们看 testCase 结构声明：
    
    ```
    type testCase struct {
        // 原 SQL
        src     string
        // 是否能被正确 parse
        ok      bool
        // 预期的 restore SQL
        restore string
    }
    ```
    
    测试代码会判断原 SQL parse 出 AST 后再还原的 SQL 是否与预期的 restore SQL 相等，具体的测试逻辑在 `parser_test.go` 中 `RunTest()`、`RunRestoreTest()` 函数，逻辑与前例类似，此处不再赘述。

编辑按：添加 TiDB Robot 微信，加入 TiDB Contributor Club，无门槛参与开源项目，改变世界从这里开始吧（萌萌哒）。

![](media/tidb-robot.jpg "tidb_rpbot")