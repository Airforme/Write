# SF_接口文档
## 1.保存造价数据
### 请求URL

> /cost/saveCostDetail.action

### 请求方式

> POST

### 参数

|  参数名   | 必选 |  类型  |   说明   |            示例            |
| -------- | --- | ------ | ------- | -------------------------- |
| costList | 是   | string | 造价数据 | [{measureUnit=计量单位1..}] |

    costList参数详解
    ```json
    [{
        measureUnit=计量单位1, 
        projectCode=清单项目编码1, 
        projectFeature=清单项目特征1, 
        hj=123.00, 
        mapUnit=长度, 
        contractId=版本号1, 
        quantity=11, 
        unitPrice=111.00, 
        tempPrice=1.00, 
        projectName=清单项目名称1, 
        dbidCode=01.01.1326.01_03.02.06.01_01.0001_XXXX
    }, 
    ...
    }]
    ```
### 返回示例

```json
{
    "msg": "接口调用成功",
    "code": "200",
    "data": "",
    "dataEncode": false
}
```

### 返回参数说明

| 参数名  |   值    |   说明   |
| ---- | ------ | ------ |
| code |  200   |  查询成功  |
|      |  502   |  参数错误  |
|      |  500   | 服务端报错  |
| msg  | 接口调用成功 | 返回结果说明 |


### 备注
