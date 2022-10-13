# 仪表盘JSON模型

Grafana仪表盘由一个JSON对象表示，该对象存储仪表盘的元数据。
仪表板元数据包括仪表板属性、面板元数据、模板变量、面板查询等。

## JSON字段

用户创建新仪表板时，将使用以下字段初始化新仪表板JSON对象：


> 注意：在下面的JSON中，id显示为null，这是在保存仪表板之前分配给它的默认值。保存仪表板后，将为id字段分配一个整数值。

```json
{
  "id": null,
  "uid": "cLV5GDCkz",
  "title": "New dashboard",
  "tags": [],
  "style": "dark",
  "timezone": "browser",
  "editable": true,
  "graphTooltip": 1,
  "panels": [],
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {
    "time_options": [],
    "refresh_intervals": []
  },
  "templating": {
    "list": []
  },
  "annotations": {
    "list": []
  },
  "refresh": "5s",
  "schemaVersion": 17,
  "version": 0,
  "links": []
}
```

下面介绍了仪表板JSON中的每个字段及其用法：

|名称|用法|
|:---:|:---:|
|id |仪表盘唯一数字标识符。（由数据库生成）。|
|uid |任何人都可以生成的唯一仪表板标识符。字符串（8-40）|
|title |当前仪表盘标题 |
|tags |与仪表盘关联的标签，字符串数组 |
|style |仪表板主题，即暗或亮 |
|timezone |仪表板的时区，即utc或浏览器 |
|editable |仪表板是否可编辑 |
|graphTooltip |0表示没有共享准线和工具提示(默认)，1表示共享准线，2表示共享准线和共享工具提示 |
|time |仪表板的时间范围，即过去6小时、过去7天等 |
|timepicker |[时间选择器](#timepicker时间选择器)元数据，有关详细信息，请参阅时间选择器部分 |
|templating |[模板化元数据](#templating模板化)，有关详细信息，请参阅模板化部分 |
|annotations |[注释](#annotations注释)元数据，有关详细信息，请参阅注释部分 |
|refresh |自动刷新间隔 |
|schemaVersion |JSON模式的版本（整数），每次Grafana更新对所述模式进行更改时递增 |
|version |仪表板版本（整数），每次更新仪表板时递增 |
|panels |面板阵列，详见下文 |

### Panels(面板)
Panels是仪表盘的组成部分。它由数据源查询、图形类型、别名等组成。Panels JSON由一个JSON对象数组组成，每个对象代表一个不同的Panels。大多数字段对于所有Panels都是通用的，但有些字段取决于Panels类型。以下是文本Panels的Panels JSON示例。

```json
"panels": [
  {
    "type": "text",
    "title": "Panel Title",
    "gridPos": {
      "x": 0,
      "y": 0,
      "w": 12,
      "h": 9
    },
    "id": 4,
    "mode": "markdown",
    "content": "# title"
  }
```

#### Panel尺寸和位置

gridPos 属性以网格坐标描述panel的尺寸和位置。

- `w` 1-24（仪表板的宽度分为24列）。
- `h` 在网格高度单位中，每个代表30个像素。
- `x` x位置，单位与w相同。
- `y` y位置，单位与h相同。

网格有负重力，如果Panel上方有空间，可以会向上移动Panel。

### timepicker(时间选择器) 

```json
"timepicker": {
    "collapse": false,
    "enable": true,
    "notice": false,
    "now": true,
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "status": "Stable",
    "type": "timepicker"
  }
```

字段的用法解释如下：

|名称|用法|
|:---:|:---:|
|collapse |timepicker是否已折叠 |
|enable |timepicker是否已启用 |
|notice |TODO |
|now |TODO |
|refresh_intervals |刷新间隔 |
|status |状态 |
|type |类型 |



### templating(模板化)

`templating` 字段包含一组模板变量及其保存的值以及一些其他元数据，例如：

```json
"templating": {
    "enable": true,
    "list": [
      {
        "allFormat": "wildcard",
        "current": {
          "tags": [],
          "text": "prod",
          "value": "prod"
        },
        "datasource": null,
        "includeAll": true,
        "name": "env",
        "options": [
          {
            "selected": false,
            "text": "All",
            "value": "*"
          },
          {
            "selected": false,
            "text": "stage",
            "value": "stage"
          },
          {
            "selected": false,
            "text": "test",
            "value": "test"
          }
        ],
        "query": "tag_values(cpu.utilization.average,env)",
        "refresh": false,
        "type": "query"
      },
      {
        "allFormat": "wildcard",
        "current": {
          "text": "apache",
          "value": "apache"
        },
        "datasource": null,
        "includeAll": false,
        "multi": false,
        "multiFormat": "glob",
        "name": "app",
        "options": [
          {
            "selected": true,
            "text": "tomcat",
            "value": "tomcat"
          },
          {
            "selected": false,
            "text": "cassandra",
            "value": "cassandra"
          }
        ],
        "query": "tag_values(cpu.utilization.average,app)",
        "refresh": false,
        "regex": "",
        "type": "query"
      }
    ]
  }
```

模板部分中上述字段的用法解释如下：

|名称|用法|
|:---:|:---:|
|enable |是否启用模板 |
|list |一个对象数组，每个对象表示一个模板变量 |
|allFormat |从数据源获取所有值时使用的格式，例如：通配符、glob、regex、管道等 |
|current |在仪表板上显示当前选定的变量文本/值 |
|data source |显示变量的数据源 |
|includeAll	|所有值选项是否可用 |
|multi |是否可以从变量值列表中选择多个值 |
|multiFormat |从数据源获取时间序列时使用的格式 |
|name |变量名称 |
|options |仪表板上可供选择的变量文本/值对数组 |
|query |用于获取变量值的数据源查询 |
|refresh |TODO |
|regex |TODO |
|type |变量类型，即自定义、查询或间隔 |


### annotations(注释)


### 参考资料

[https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/)