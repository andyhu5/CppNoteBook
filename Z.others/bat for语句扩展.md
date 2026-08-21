在 Windows 批处理脚本中，for 语句是一个强大的工具，用于循环处理文件、字符串、数字范围等。结合参数扩展和变量操作，for 可以实现更复杂的功能。以下是 for 语句的基本用法、扩展形式及其详细解释。 

------

1. 基本 for 语句

- **功能**: 遍历一组项目（文件、字符串等）并执行命令。

- **语法**:

  cmd

  ```text
  for %%variable in (set) do command
  ```

  - %%variable: 循环变量（在脚本中用 %%a, %%b 等；在命令行直接输入时用单 %a）。
  - (set): 要遍历的集合（文件、字符串、数字等）。
  - command: 每次迭代执行的命令。

- **示例**:

  cmd

  ```text
  @echo off
  for %%i in (apple banana orange) do (
    echo Fruit: %%i
  )
  pause
  ```

  - **输出**:

    ```text
    Fruit: apple
    Fruit: banana
    Fruit: orange
    Press any key to continue . . .
    ```

  - **解释**: 遍历字符串列表 apple banana orange，每次迭代将值赋给 %%i 并输出。

------

2. for 语句的扩展形式

批处理的 for 命令有多种变体，通过选项和参数扩展实现更复杂的功能。以下是主要形式：

(1) for /l - 遍历数字范围

- **功能**: 按数字范围循环，适合计数或步进操作。

- **语法**:

  cmd

  ```text
  for /l %%variable in (start,step,end) do command
  ```

  - start: 起始值（整数）。
  - step: 每次循环的增量（正数递增，负数递减）。
  - end: 结束值（包含在内）。

- **示例**:

  cmd

  ```text
  @echo off
  for /l %%i in (1,2,5) do (
    echo Number: %%i
  )
  pause
  ```

  - **输出**:

    ```text
    Number: 1
    Number: 3
    Number: 5
    Press any key to continue . . .
    ```

  - **解释**:

    - 从 1 开始，每次加 2，直到 5（包含）。
    - 结果为 1, 3, 5。

- **用途**: 计数器、生成序号。

(2) for /f - 解析文件或命令输出

- **功能**: 解析文本文件内容、字符串或命令输出，逐行处理。

- **语法**:

  cmd

  ```text
  for /f "options" %%variable in (source) do command
  ```

  - options: 控制解析行为的参数（见下文）。
  - source: 可以是文件、字符串或命令（用单引号或反引号括起来）。

- **常见选项**:

  - delims=xxx: 指定分隔符（默认是空格和制表符）。
  - tokens=n,m: 选择第 n 和第 m 个字段（默认 tokens=1）。
  - skip=n: 跳过前 n 行。
  - eol=c: 设置行结束字符（忽略以 c 开头的行）。
  - usebackq: 允许用反引号（`command`）执行命令，用单引号处理字符串，双引号读文件。

- **示例 1: 解析字符串**

  cmd

  ```text
  @echo off
  for /f "tokens=2 delims=," %%i in ("apple,banana,orange") do (
    echo Second item: %%i
  )
  pause
  ```

  - **输出**:

    ```text
    Second item: banana
    Press any key to continue . . .
    ```

  - **解释**:

    - delims=,: 以逗号分隔。
    - tokens=2: 取第二个字段（banana）。

- **示例 2: 解析文件**

  - 假设有个文件 data.txt 内容：

    ```text
    apple 10 red
    banana 20 yellow
    orange 30 orange
    ```

  - 脚本：

    cmd

    ```text
    @echo off
    for /f "tokens=1,3" %%a in (data.txt) do (
      echo Fruit: %%a, Color: %%b
    )
    pause
    ```

  - **输出**:

    ```text
    Fruit: apple, Color: red
    Fruit: banana, Color: yellow
    Fruit: orange, Color: orange
    Press any key to continue . . .
    ```

  - **解释**:

    - tokens=1,3: 取每行的第 1 和第 3 个字段，赋给 %%a 和 %%b。
    - 默认分隔符是空格。

- **示例 3: 解析命令输出**

  cmd

  ```text
  @echo off
  for /f "tokens=1" %%i in ('dir') do (
    echo Item: %%i
  )
  pause
  ```

  - **输出**:

    ```text
    Item: dir
    Item: dir
    ...
    Press any key to continue . . .
    ```

  - **解释**:

    - dir 命令输出文件和目录列表。
    - tokens=1: 取每行的第一个字段（可能包含日期或文件名，具体取决于 dir 输出格式）。

- **usebackq 示例**:

  cmd

  ```text
  @echo off
  for /f "usebackq tokens=1" %%i in (`echo Hello World`) do (
    echo First word: %%i
  )
  pause
  ```

  - **输出**:

    ```text
    First word: Hello
    Press any key to continue . . .
    ```

  - **解释**:

    - usebackq: 允许用反引号执行命令。
    - tokens=1: 取命令 echo Hello World 输出的第一个字段。

(3) for /r - 递归遍历目录

- **功能**: 遍历指定目录及其子目录中的所有文件。

- **语法**:

  cmd

  ```text
  for /r [path] %%variable in (pattern) do command
  ```

  - path: 起始目录（默认当前目录）。
  - pattern: 文件匹配模式（如 *.txt）。

- **示例**:

  cmd

  ```text
  @echo off
  for /r "%~dp0" %%f in (*.txt) do (
    echo Found file: %%f
  )
  pause
  ```

  - **输出**:

    ```text
    Found file: C:\Scripts\data.txt
    Found file: C:\Scripts\subfolder\test.txt
    Press any key to continue . . .
    ```

  - **解释**:

    - /r "%~dp0": 从脚本所在目录递归搜索。
    - *.txt: 匹配所有 .txt 文件。
    - %%f: 每次迭代包含文件的完整路径。

(4) for /d - 遍历目录

- **功能**: 遍历指定目录中的子目录（不包括文件）。

- **语法**:

  cmd

  ```text
  for /d %%variable in (pattern) do command
  ```

  - pattern: 目录匹配模式（如 * 表示所有目录）。

- **示例**:

  cmd

  ```text
  @echo off
  for /d %%d in (*) do (
    echo Directory: %%d
  )
  pause
  ```

  - **输出**:

    ```text
    Directory: subfolder1
    Directory: subfolder2
    Press any key to continue . . .
    ```

  - **解释**:

    - (*): 匹配当前目录下的所有子目录。
    - %%d: 每次迭代包含目录名（不含路径）。

- **结合 /r**:

  cmd

  ```text
  @echo off
  for /r /d %%d in (*) do (
    echo Directory: %%d
  )
  pause
  ```

  - **输出**: 列出当前目录及子目录的所有目录的完整路径。

------

3. 结合参数扩展

批处理的 for 循环常与参数扩展结合，处理路径、文件名等：

- **常见扩展**:

  - %%~fI: 完整路径。
  - %%~nI: 仅文件名（无扩展名）。
  - %%~xI: 仅扩展名。
  - %%~dpI: 驱动器和路径（带尾随反斜杠）。
  - %%~nxI: 文件名和扩展名。

- **示例**:

  cmd

  ```text
  @echo off
  for %%f in (*.txt) do (
    echo Full path: %%~ff
    echo Name: %%~nf
    echo Extension: %%~xf
    echo Directory: %%~dpf
  )
  pause
  ```

  - **输出**（假设有 data.txt）:

    ```text
    Full path: C:\Scripts\data.txt
    Name: data
    Extension: .txt
    Directory: C:\Scripts\
    Press any key to continue . . .
    ```

------

4. 注意事项

- **循环变量**:

  - 在脚本中用 %%a, %%b 等；在命令行用 %a, %b。
  - 变量名大小写敏感（%%I 和 %%i 不同）。

- **延迟扩展**:

  - 在循环中修改变量时，需用 setlocal EnableDelayedExpansion 和 !var!：

    cmd

    ```text
    @echo off
    setlocal EnableDelayedExpansion
    set "count=0"
    for %%i in (a b c) do (
      set /a count+=1
      echo Item %%i, Count: !count!
    )
    endlocal
    pause
    ```

    - **输出**:

      ```text
      Item a, Count: 1
      Item b, Count: 2
      Item c, Count: 3
      Press any key to continue . . .
      ```

- **特殊字符**:

  - 文件名或字符串含空格、& 等，需用引号（"%%f"）。

- **性能**:

  - 批处理不适合处理大量数据，复杂任务考虑 PowerShell。

------

5. 总结

- **基本 for**: 遍历字符串或文件（for %%i in (set) do）。
- **扩展形式**:
  - /l: 数字范围循环。
  - /f: 解析文件或命令输出。
  - /r: 递归遍历文件。
  - /d: 遍历目录。
- **参数扩展**: 如 %%~nf 提取文件名，%%~dpI 提取路径。
- **用途**: 处理文件、计数、解析文本、遍历目录等。

如果你有具体需求（例如遍历特定文件、解析日志），告诉我，我可以帮你写脚本！