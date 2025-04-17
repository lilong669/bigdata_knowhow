# 自动化建表语句

```python
# -*- coding: utf-8 -*-
import jaydebeapi
from datetime import datetime
import logging
import argparse

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
app = logging.getLogger('DWDTableGenerator')

# 建表脚本模板，定义在开头，易于修改
TABLE_SCRIPT_TEMPLATE = """-- model_name: {dwd_table_name}
-- descript: {table_comment}
-- date: {current_date}
-- author: lilong

set character.literal.as.string = true;
set argodb.dynamic.create.partition.enabled = true;
set hive.exec.dynamic.partition = true;
set stargate.dynamic.partition.enabled = true;
set hive.default.fileformat = holodesk;

drop table if exists {dwd_table_name};

create table if not exists {dwd_table_name} (
{fields}
) 
comment '{table_comment}'
stored as holodesk;

insert into {dwd_table_name}
select 
{insert_fields}
from {ods_table_name};
"""

# 数据库连接配置
driver_dicts = {
    "inceptor": {
        "driver": "jdbc:hive2://10.149.177.208:10000/default",
        "driver_class": "org.apache.hive.jdbc.HiveDriver",
        "jar_path": "/root/my_py/drivers/inceptor-driver-8.37.3.jar",
        "url": "jdbc:hive2://10.149.177.208:10000/default",
        "username": "hive",
        "password": "123456"
    }
}

def get_table_comment(table_name):
    """
    使用 DESCRIBE FORMATTED 获取表注释
    """
    driver_class = driver_dicts["inceptor"]["driver_class"]
    url = driver_dicts["inceptor"]["url"]
    username = driver_dicts["inceptor"]["username"]
    password = driver_dicts["inceptor"]["password"]
    jar_path = driver_dicts["inceptor"]["jar_path"]

    table_comment = ""
    try:
        app.info(f"获取表 {table_name} 的注释")
        conn = jaydebeapi.connect(
            jclassname=driver_class,
            url=url,
            driver_args=[username, password],
            jars=jar_path
        )
        app.info("成功连接到 Inceptor！")
        
        query = f"DESCRIBE FORMATTED {table_name}"
        app.info(f"执行查询: {query}")
        curs = conn.cursor()
        curs.execute(query)
        results = curs.fetchall()
        
        # 解析 DESCRIBE FORMATTED 的结果，查找 comment
        for row in results:
            if len(row) >= 2 and row[0] and "comment" in row[0].lower():
                table_comment = row[1].strip() if row[1] else ""
                break
        
        curs.close()
        conn.close()
        app.info("连接已关闭。")
    
    except Exception as e:
        app.error(f"获取表注释失败: {e}")
    
    return table_comment

def get_table_schema(table_name):
    """
    使用 DESC 获取表结构，排除 # 开头行及其后续内容
    """
    driver_class = driver_dicts["inceptor"]["driver_class"]
    url = driver_dicts["inceptor"]["url"]
    username = driver_dicts["inceptor"]["username"]
    password = driver_dicts["inceptor"]["password"]
    jar_path = driver_dicts["inceptor"]["jar_path"]

    schema = []
    try:
        app.info(f"获取表 {table_name} 的结构")
        conn = jaydebeapi.connect(
            jclassname=driver_class,
            url=url,
            driver_args=[username, password],
            jars=jar_path
        )
        app.info("成功连接到 Inceptor！")
        
        query = f"DESC {table_name}"
        app.info(f"执行查询: {query}")
        curs = conn.cursor()
        curs.execute(query)
        results = curs.fetchall()
        
        # 解析 DESC 的结果，提取 col_name, data_type, comment
        skip = False
        for row in results:
            if len(row) >= 6:  # 确保有足够的列
                col_name = row[0].strip()
                # 遇到 # 开头的行，标记跳过后续行
                if col_name.startswith('#'):
                    skip = True
                    continue
                # 跳过 # 之后的行
                if skip:
                    continue
                # 处理有效字段行
                if col_name:  # 确保 col_name 不为空
                    data_type = row[1].strip()
                    comment = row[5].strip() if row[5] else col_name  # 使用 col_name 作为默认注释
                    schema.append({
                        "column_name": col_name,
                        "data_type": data_type,
                        "comment": comment
                    })
        
        curs.close()
        conn.close()
        app.info("连接已关闭。")
    
    except Exception as e:
        app.error(f"获取表结构失败: {e}")
    
    return schema

def generate_dwd_table_script(ods_table_name, dwd_table_name):
    """
    根据 ODS 表名和指定的 DWD 表名生成建表脚本
    """
    # 获取表结构和表注释
    schema = get_table_schema(ods_table_name)
    table_comment = get_table_comment(ods_table_name)
    
    if not schema:
        app.error("表结构获取失败，无法生成脚本")
        return None
    
    # 如果表注释为空，使用短表名作为默认注释
    table_short_name = ods_table_name.split('.')[-1]
    table_comment = table_comment or f"{table_short_name} DWD 数据模型"
    
    # 当前日期
    current_date = datetime.now().strftime("%Y%m%d")
    
    # 生成字段定义
    fields = []
    for col in schema:
        col_name = col["column_name"]
        col_type = col["data_type"]
        col_comment = col["comment"]
        fields.append(f"    {col_name} {col_type} comment '{col_comment}'")
    fields_str = ",\n".join(fields)
    
    # 生成 INSERT 语句的字段，带注释，格式为 字段, -- 字段注释，最后一个字段无逗号
    insert_fields = []
    for i, col in enumerate(schema):
        col_name = col["column_name"]
        col_comment = col["comment"]
        # 对 decimal 类型进行转换
        if col["data_type"].startswith("decimal"):
            field = f"cast({col_name} as {col['data_type']}) as {col_name}"
        else:
            field = col_name
        # 最后一个字段不加逗号
        if i == len(schema) - 1:
            insert_fields.append(f"    {field} -- {col_comment}")
        else:
            insert_fields.append(f"    {field}, -- {col_comment}")
    insert_fields_str = "\n".join(insert_fields)
    
    # 使用模板生成脚本
    script = TABLE_SCRIPT_TEMPLATE.format(
        dwd_table_name=dwd_table_name,
        table_comment=table_comment,
        current_date=current_date,
        fields=fields_str,
        insert_fields=insert_fields_str,
        ods_table_name=ods_table_name
    )
    
    return script

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="生成 DWD 模型建表脚本")
    parser.add_argument("--ods_table", required=True, help="ODS 表名（例如 ods.table_name）")
    parser.add_argument("--dwd_table", required=True, help="DWD 表名（例如 dwd.table_name）")
    
    args = parser.parse_args()
    
    ods_table_name = args.ods_table
    dwd_table_name = args.dwd_table
    
    app.info(f"开始生成建表脚本，ODS 表: {ods_table_name}, DWD 表: {dwd_table_name}")
    
    script = generate_dwd_table_script(ods_table_name, dwd_table_name)
    if script:
        print(script)
    else:
        app.error("建表脚本生成失败")
        print("生成建表脚本失败")
```
