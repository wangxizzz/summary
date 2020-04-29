## es整合springdata API相关
Spring Data 的另一个强大功能，是根据方法名称自动实现功能。

比如：你的方法名叫做：findByTitle，那么它就知道你是根据title查询，然后自动帮你完成，无需写实现类。

当然，方法名称要符合一定的约定：
<table>
<thead>
<tr>
<th>Keyword</th>
<th>Sample</th>
<th>Elasticsearch Query String</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>And</code></td>
<td><code>findByNameAndPrice</code></td>
<td><code>{"bool" : {"must" : [ {"field" : {"name" : "?"}}, {"field" : {"price" : "?"}} ]}}</code></td>
</tr>
<tr>
<td><code>Or</code></td>
<td><code>findByNameOrPrice</code></td>
<td><code>{"bool" : {"should" : [ {"field" : {"name" : "?"}}, {"field" : {"price" : "?"}} ]}}</code></td>
</tr>
<tr>
<td><code>Is</code></td>
<td><code>findByName</code></td>
<td><code>{"bool" : {"must" : {"field" : {"name" : "?"}}}}</code></td>
</tr>
<tr>
<td><code>Not</code></td>
<td><code>findByNameNot</code></td>
<td><code>{"bool" : {"must_not" : {"field" : {"name" : "?"}}}}</code></td>
</tr>
<tr>
<td><code>Between</code></td>
<td><code>findByPriceBetween</code></td>
<td><code>{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}</code></td>
</tr>
<tr>
<td><code>LessThanEqual</code></td>
<td><code>findByPriceLessThan</code></td>
<td><code>{"bool" : {"must" : {"range" : {"price" : {"from" : null,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}</code></td>
</tr>
<tr>
<td><code>GreaterThanEqual</code></td>
<td><code>findByPriceGreaterThan</code></td>
<td><code>{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : null,"include_lower" : true,"include_upper" : true}}}}}</code></td>
</tr>
<tr>
<td><code>Before</code></td>
<td><code>findByPriceBefore</code></td>
<td><code>{"bool" : {"must" : {"range" : {"price" : {"from" : null,"to" : ?,"include_lower" : true,"include_upper" : true}}}}}</code></td>
</tr>
<tr>
<td><code>After</code></td>
<td><code>findByPriceAfter</code></td>
<td><code>{"bool" : {"must" : {"range" : {"price" : {"from" : ?,"to" : null,"include_lower" : true,"include_upper" : true}}}}}</code></td>
</tr>
<tr>
<td><code>Like</code></td>
<td><code>findByNameLike</code></td>
<td><code>{"bool" : {"must" : {"field" : {"name" : {"query" : "?*","analyze_wildcard" : true}}}}}</code></td>
</tr>
<tr>
<td><code>StartingWith</code></td>
<td><code>findByNameStartingWith</code></td>
<td><code>{"bool" : {"must" : {"field" : {"name" : {"query" : "?*","analyze_wildcard" : true}}}}}</code></td>
</tr>
<tr>
<td><code>EndingWith</code></td>
<td><code>findByNameEndingWith</code></td>
<td><code>{"bool" : {"must" : {"field" : {"name" : {"query" : "*?","analyze_wildcard" : true}}}}}</code></td>
</tr>
<tr>
<td><code>Contains/Containing</code></td>
<td><code>findByNameContaining</code></td>
<td><code>{"bool" : {"must" : {"field" : {"name" : {"query" : "**?**","analyze_wildcard" : true}}}}}</code></td>
</tr>
<tr>
<td><code>In</code></td>
<td><code>findByNameIn(Collection&lt;String&gt;names)</code></td>
<td><code>{"bool" : {"must" : {"bool" : {"should" : [ {"field" : {"name" : "?"}}, {"field" : {"name" : "?"}} ]}}}}</code></td>
</tr>
<tr>
<td><code>NotIn</code></td>
<td><code>findByNameNotIn(Collection&lt;String&gt;names)</code></td>
<td><code>{"bool" : {"must_not" : {"bool" : {"should" : {"field" : {"name" : "?"}}}}}}</code></td>
</tr>
<tr>
<td><code>Near</code></td>
<td><code>findByStoreNear</code></td>
<td><code>Not Supported Yet !</code></td>
</tr>
<tr>
<td><code>True</code></td>
<td><code>findByAvailableTrue</code></td>
<td><code>{"bool" : {"must" : {"field" : {"available" : true}}}}</code></td>
</tr>
<tr>
<td><code>False</code></td>
<td><code>findByAvailableFalse</code></td>
<td><code>{"bool" : {"must" : {"field" : {"available" : false}}}}</code></td>
</tr>
<tr>
<td><code>OrderBy</code></td>
<td><code>findByAvailableTrueOrderByNameDesc</code></td>
<td><code>{"sort" : [{ "name" : {"order" : "desc"} }],"bool" : {"must" : {"field" : {"available" : true}}}}</code></td>
</tr>
</tbody>
</table>

## Java High Level REST Client
参考网址：https://www.jianshu.com/p/5cb91ed22956