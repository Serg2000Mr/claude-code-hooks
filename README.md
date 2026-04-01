# claude-code-hooks

Хуки [Claude Code](https://claude.ai/code) для Windows — звуковые уведомления, защита файлов, автоматизация.

<details>
<summary><h3>1. Звук при запросе разрешения</h3></summary>

Когда Claude Code запрашивает разрешение на выполнение команды, играет системный звук Windows. Удобно если вы отвлеклись — не нужно следить за экраном.

Добавьте в `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "python3 -c \"import sys,json,winsound; d=json.load(sys.stdin); d.get('permission_suggestions') and winsound.PlaySound('C:/Windows/Media/Windows Unlock.wav',winsound.SND_FILENAME)\""
          }
        ]
      }
    ]
  }
}
```

### Как это работает

Событие `PermissionRequest` срабатывает перед каждой проверкой разрешения. Чтобы звук играл только когда диалог реально показывается пользователю (а не при авто-разрешённых командах), хук проверяет поле `permission_suggestions` во входном JSON.

- `permission_suggestions` непустой → диалог появится → **звук**
- `permission_suggestions` пустой → авто-разрешено (например, Edit в режиме acceptEdits) → **тишина**

### Почему winsound, а не PowerShell

`winsound` — стандартный модуль Python для Windows, работает без subprocess. Альтернативные подходы не работают:

| Подход | Проблема |
|--------|----------|
| `powershell ... PlaySync()` через bash | stdin не передаётся в дочерний процесс |
| `shell: powershell` + `$input` | stdin недоступен в command-hook |
| `subprocess.Popen(["powershell", ...])` | Popen неблокирующий; при `async:true` stdin не пишется |
| `Notification: permission_prompt` | Событие не срабатывает в VS Code расширении |

### Отладка

Чтобы понять какие события реально приходят, временно добавьте лог-хук:

```json
"PermissionRequest": [
  {
    "matcher": "",
    "hooks": [
      {
        "type": "command",
        "command": "python3 -c \"import sys,json,datetime; d=json.load(sys.stdin); open('/tmp/hook-debug.log','a').write(datetime.datetime.now().isoformat()+' '+json.dumps(d)+'\\n')\""
      }
    ]
  }
]
```

Запустите авто-разрешённую и неразрешённую команду, прочитайте лог — увидите разницу в `permission_suggestions`.

</details>

<details>
<summary><h3>2. Защита файлов вне диска C:</h3></summary>

Блокирует любое чтение или запись файлов на дисках кроме C:. Полезно если Claude Code работает в нескольких проектах и не должен случайно трогать другие разделы.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Edit|Write|Glob|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "python3 -c \"import sys,json; d=json.load(sys.stdin); ti=d.get('tool_input',{}); p=ti.get('file_path') or ti.get('pattern') or ti.get('path') or ''; p[1:2]==':' and p[0].lower()!='c' and print(json.dumps({'hookSpecificOutput':{'hookEventName':'PreToolUse','permissionDecision':'deny','permissionDecisionReason':'Доступ вне диска C: запрещён'}}))\""
          }
        ]
      }
    ]
  }
}
```

</details>

## Требования

- Windows 10/11
- Python 3 (доступен как `python3` в Git Bash)
- Claude Code (VS Code extension или CLI)
