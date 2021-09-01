# JSQLParser的简单使用
这里简单记录一下最近了解到的 JSQLParser 这个 SQL 语句解析器的用法，方便有需要的时候参考。

#### 正文：

    import net.sf.jsqlparser.JSQLParserException;
    import net.sf.jsqlparser.parser.CCJSqlParserUtil;
    import net.sf.jsqlparser.schema.Table;
    import net.sf.jsqlparser.statement.Statement;
    import net.sf.jsqlparser.statement.select.*;
    import net.sf.jsqlparser.util.TablesNamesFinder;

    import java.util.ArrayList;
    import java.util.List;

    public class JSQLParserTest {
        public static void main(String[] args) {
            String sql = "SELECT\n" +
                    "  a.id,\n" +
                    "  username,\n" +
                    "  b.workflow_name,\n" +
                    "  reviewok_time\n" +
                    "FROM archer.sql_users a\n" +
                    "  JOIN sql_workflow b ON a.username = b.engineer\n" +
                    "  JOIN archive.table3 c ON a.username = c.col5;\n";

            try {
                Statement stmt = CCJSqlParserUtil.parse(sql);
                Select selectStatement = (Select) stmt;

                TablesNamesFinder tablesNamesFinder = new TablesNamesFinder();
                List<String> tableList = tablesNamesFinder.getTableList(selectStatement);
                for (String name: tableList) {
                    System.out.println("table: " + name);
                }
                System.out.println();

                List<String> columnList = new ArrayList<String>();
                PlainSelect ps = (PlainSelect) selectStatement.getSelectBody();
                List<SelectItem> selectItems = ps.getSelectItems();
                // System.out.println(selectItems);
                selectItems.stream().forEach(selectItem -> columnList.add(selectItem.toString()));
                for (String name: columnList) {
                    System.out.println("column: " + name);
                }
                System.out.println();

                FromItem fromItem = ps.getFromItem();
                Table table = (Table) fromItem;
                System.out.println(table.getName() + ":\t" + table.getAlias());

                List<Join> joins = ps.getJoins();
                for (Join join : joins) {
                    fromItem = join.getRightItem();
                    table = (Table) fromItem;
                    System.out.println(table.getName() + ":\t" + table.getAlias());
                }

            } catch (JSQLParserException e) {
                e.printStackTrace();
            }
        }

    }

简单来说就是，JSQLParser 使用起来还算简单，但就上面的测试 SQL 来看，效果难说完美（表名提取那里会把 archer.sql_users 里面的「archer.」给略掉，只剩下 sql_users）。

据说「[General Sql Parser’s (GSP)](http://www.sqlparser.com/)」的效果很好，不过是付费的（想来在很多方面要想把一件事情做好，还是需要专门的人力支持、维护，付费是有付费的道理的），还没有机会测试，以后有机会了再试试效果。

##### 参考链接：

[https://github.com/JSQLParser/JSqlParser/wiki/Examples-of-SQL-parsing](https://github.com/JSQLParser/JSqlParser/wiki/Examples-of-SQL-parsing)

Jsqlparser 使用  
[https://www.yuech.net/2015/10/09/Jsqlparser%E4%BD%BF%E7%94%A8/](https://www.yuech.net/2015/10/09/Jsqlparser%E4%BD%BF%E7%94%A8/)

java-How to parse sql columns with JDBC or jSqlParser ?  
[https://www.bswen.com/2019/05/android-How-to-parse-sql-columns-with-JDBC-or-jSqlParser.html](https://www.bswen.com/2019/05/android-How-to-parse-sql-columns-with-JDBC-or-jSqlParser.html)

使用 java sql parser 插件 Jsqlparser 实例 (一)  
[https://blog.csdn.net/u014297722/article/details/53256533](https://blog.csdn.net/u014297722/article/details/53256533)

Parsing table and column names from SQL/HQL Java  
[https://stackoverflow.com/questions/40908062/parsing-table-and-column-names-from-sql-hql-java](https://stackoverflow.com/questions/40908062/parsing-table-and-column-names-from-sql-hql-java)

[https://stackoverflow.com/questions/16768365/how-to-retrieve-table-and-column-names-from-sql-using-jsqlparse](https://stackoverflow.com/questions/16768365/how-to-retrieve-table-and-column-names-from-sql-using-jsqlparse)

[https://www.programcreek.com/java-api-examples/index.php?api=net.sf.jsqlparser.schema.Column](https://www.programcreek.com/java-api-examples/index.php?api=net.sf.jsqlparser.schema.Column)  
[https://www.programcreek.com/java-api-examples/?api=net.sf.jsqlparser.statement.select.AllTableColumns](https://www.programcreek.com/java-api-examples/?api=net.sf.jsqlparser.statement.select.AllTableColumns)  
[https://vimsky.com/examples/detail/java-class-net.sf.jsqlparser.schema.Table.html](https://vimsky.com/examples/detail/java-class-net.sf.jsqlparser.schema.Table.html)

[https://stackoverflow.com/questions/37250588/jsqlparser-getting-table-name-from-column](https://stackoverflow.com/questions/37250588/jsqlparser-getting-table-name-from-column)

JSQLParser 来分析复杂 SQL，实现 UI 业务一次 SQL 搞定  
[https://www.jianshu.com/p/f57bc22b5b32](https://www.jianshu.com/p/f57bc22b5b32)

JPA 表租户 SQL 解析实现  
[https://www.codenong.com/js8ad3b4d6bd43/](https://www.codenong.com/js8ad3b4d6bd43/)

几种基于 Java 的 SQL 解析工具的比较与调用  
[https://blog.csdn.net/qq_21383435/article/details/81984297](https://blog.csdn.net/qq_21383435/article/details/81984297)

基于 Java 的 SQL 解析工具的比较与学习  
[https://blog.csdn.net/czq850114000/article/details/80844689](https://blog.csdn.net/czq850114000/article/details/80844689) 
 [https://ixyzero.com/blog/archives/4806.html](https://ixyzero.com/blog/archives/4806.html)
