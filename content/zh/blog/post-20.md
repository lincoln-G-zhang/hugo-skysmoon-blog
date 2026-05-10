+++
title = "替换字符串"
slug = "post-20"
date = "2025-04-05T13:39:00+08:00"
description = "替换指定字符串"
categories = ["shell"]
+++

# 替换指定字符串

------------

```shell
#!/bin/bash

# 检查参数数量（至少 3 个参数：目录、要替换的字符串、替换后的字符串，后跟文件列表）
if [ $# -lt 3 ]; then
    echo "用法: $0 <目录路径> <要替换的字符串> <替换后的字符串> [文件1] [文件2] ..."
    echo "示例: $0 ./dir 'old string' 'new string' file1.txt file2.conf"
    exit 1
fi

# 解析参数
directory="$1"
old_string="$2"
new_string="$3"
shift 3  # 移除前三个参数，剩下的为文件列表

timestamp=$(date +%Y%m%d%H%M%S)

# 处理每个指定的文件
for file in "$@"; do
    full_path="${directory}/${file}"  # 拼接完整路径
    if [ ! -f "$full_path" ]; then
        echo "错误：文件 $full_path 不存在或不是普通文件"
        continue
    fi

    # 备份文件（以时间戳结尾）
    backup_file="${full_path}_${timestamp}"
    cp "$full_path" "$backup_file"
    echo "已备份文件：$full_path → $backup_file"

    # 替换字符串（处理特殊字符，使用 sed 的 -e 选项并转义特殊字符）
    # 注意：这里使用双引号包裹 sed 命令，以便 $old_string 和 $new_string 中的变量被正确解析
    # 对特殊字符（如引号、分号）进行转义，避免 sed 语法错误
    escaped_old=$(echo "$old_string" | sed 's/[&\/]/\\&/g; s/[^[:alnum:]]/\\&/g')
    escaped_new=$(echo "$new_string" | sed 's/[&\/]/\\&/g; s/[^[:alnum:]]/\\&/g')
    sed -i -e "s|${escaped_old}|${escaped_new}|g" "$full_path"
    echo "已完成替换：$full_path"

    # 验证：打印替换前后的前 5 行（标记替换内容）
    echo "------------------------"
    echo "验证文件：$full_path"
    echo "替换前（备份文件）："
    head -n 5 "$backup_file" | sed -e "s|${escaped_old}|[OLD: ${old_string}]|g"
    echo "替换后："
    head -n 5 "$full_path" | sed -e "s|${escaped_new}|[NEW: ${new_string}]|g"
    echo "------------------------"
done

echo "所有指定文件处理完成!"
```