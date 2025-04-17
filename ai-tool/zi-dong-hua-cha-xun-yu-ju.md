# 自动化查询语句

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
app = logging.getLogger('SelectQueryGenerator')

# 查询脚本模板，定义在开头，易于修改
SELECT_SCRIPT_TEMPLATE = """-- 查询表: {table_name}
-- 生成时间: {current_date}
-- 描述: 查询表 {table_name} 的所有字段

select
{fields}
from {table_name}
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

def generate_select_script(table_name):
    """
    根据表名生成 SELECT 查询脚本，字段格式为 字段, -- 字段注释，最后一个字段无逗号
    """
    # 获取表结构
    schema = get_table_schema(table_name)
    
    if not schema:
        app.error("表结构获取失败，无法生成脚本")
        return None
    
    # 当前日期
    current_date = datetime.now().strftime("%Y%m%d %H:%M:%S")
    
    # 生成 SELECT 字段，格式为 字段, -- 字段注释，最后一个字段无逗号
    fields = []
    for i, col in enumerate(schema):
        col_name = col["column_name"]
        col_comment = col["comment"]
        # 最后一个字段不加逗号
        if i == len(schema) - 1:
            fields.append(f"    {col_name} -- {col_comment}")
        else:
            fields.append(f"    {col_name}, -- {col_comment}")
    fields_str = "\n".join(fields)
    
    # 使用模板生成脚本
    script = SELECT_SCRIPT_TEMPLATE.format(
        table_name=table_name,
        current_date=current_date,
        fields=fields_str
    )
    
    return script

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="生成表的所有字段 SELECT 查询脚本")
    parser.add_argument("--table", required=True, help="表名（例如 ods.table_name）")
    
    args = parser.parse_args()
    
    table_name = args.table
    
    app.info(f"开始生成 SELECT 查询脚本，表名: {table_name}")
    
    script = generate_select_script(table_name)
    if script:
        print(script)
    else:
        app.error("SELECT 查询脚本生成失败")
        print("生成 SELECT 查询脚本失败")
```
