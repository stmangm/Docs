---
order: 1
title: Арх видение
---

[tabs]

[tab:Вкладка]

Первый

[/tab]

[tab:Вкладка]

Второй

[/tab]

[tab:Вкладка]

Третий

[/tab]

[/tabs]

|   |   |   |   |
|---|---|---|---|
|   |   |   |   |
|   |   |   |   |

:::lab 

Note

:::

:::info 



:::

:::danger 



:::

* [ ] 1

* [ ] 2

* [ ] 3

* [ ] 4

* [ ] 5

* [ ] 

   ---

```python
import sublime
import sublime_plugin
import re
import codecs


class ParceRsLogCommand(sublime_plugin.TextCommand):
    def run(self, edit):
        #print (sublime.active_window().active_view().file_name())
        file = sublime.active_window().active_view().file_name()
        if not file.endswith('_trace.log'):
            sublime.error_message(
                'Файл не является файлом трассировки RS Bank, парсинг невозможен!')
            return
        l = ''
        v = []
        block = False
        nC = 0
        nPos = 0
        def rstosql(list):
            LStr = ["'"]
            lDat = ['to_date(', 'dd.mm.yyyy', ')', 'dd.mm.yyyy hh24:mi:ss']
            LChar = ['CHR(', ')']
            try:
                list[4]
            except IndexError:
                list.insert(4, '&var_is_null')
                return list
            if list[2] == 'RSDSHORT,' or list[2] == 'RSDLONG,' or list[2] == 'RSDLPSTR,':
                list[4] = LStr[0] + list[4] + LStr[0]
            elif list[2] == 'RSDDATE,':
                if list[4] == '0.0.0':
                    list[4] = '01.01.0001'
                list[4] = lDat[0] + LStr[0] + list[4] + LStr[0] + \
                    ',' + LStr[0] + lDat[1] + LStr[0] + lDat[2]
            elif list[2] == 'RSDCHAR,':
                list[4] = LChar[0] + list[4] + LChar[1]
            elif list[2] == 'RSDTIMESTAMP,':
                list[4] = lDat[0] + LStr[0] + list[4] + ' ' + list[5] + \
                    LStr[0] + ',' + LStr[0] + lDat[3] + LStr[0] + lDat[2]
            elif list[2] == 'RSDTIME,':
                if list[4] == '0:0:0':
                    list[4] = '01.01.0001 00:00:00'
                else:
                    list[4] = '01.01.0001 ' + list[4]
                list[4] = lDat[0] + LStr[0] + list[4] + LStr[0] + \
                    ', ' + LStr[0] + lDat[3] + LStr[0] + lDat[2]
            else:
                pass
            return list
        with codecs.open(file, 'r', 'CP866') as fo:
            sublime.active_window().new_file()
            for line in fo:
                nC = nC + 1
                if re.findall('..:..:...... ..-..-....', line):  # отрезаем дату и время лога
                    line = line[:-25]
                line = ' '.join(line.split())
                line.strip()
                if re.findall('#', line) and len(line) > 0:  # value block start
                    # приведем к RS
                    tmplist = rstosql(line.split())
                    v.append(tmplist)
                    block = True
                elif not re.findall('Result=', line) and block:  # select/begin block
                    l = l + line
                elif re.findall('Result=', line):
                    c = len(v)
                    for i in range(c):  # замена значений запроса
                        l = l.replace(
                            '?', ' ' + v[i][4] + '/*' + v[i][0] + '*/', 1)
                    block = False
                    for val in v:  # вывод значений запроса
                        sublime.active_window().active_view().insert(edit, nPos, str(val) + '\n')
                        nPos = nPos + len(str(val) + '\n')
                    sublime.active_window().active_view().insert(
                        edit, nPos, "/*номер строки с тегом Result= запроса: " + str(nC) + "*/" + '\n')
                    nPos = nPos + len("/*номер строки с тегом Result= запроса: " + str(nC) + "*/" + '\n')
                    sublime.active_window().active_view().insert(edit, nPos, str(l) + '\n')
                    nPos = nPos + len(str(l) + '\n')
                    v.clear()
                    l = ''
                    sublime.active_window().active_view().insert(
                        edit, nPos, "-------------------" + '\n')
                    nPos = nPos + len("-------------------" + '\n')
if __name__ == '__main__':
    sublime.run_command('parce_rs_log')
    # sublime.run_command('parce_rs_log', {'file': 'C:\\Users\\user\\Desktop\\test.log'})
    
```