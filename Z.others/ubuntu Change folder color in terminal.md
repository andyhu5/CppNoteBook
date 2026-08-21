## Change folder color in **terminal (`ls`)**

add to ~/.bashrc

```bash
export LS_COLORS=$LS_COLORS:'di=1;33'
```



Folders are controlled by `LS_COLORS`.

```
LS_COLORS="di=1;34" ls
```

- `di` = directory
- `1;34` = **bold blue**

Other common colors:

| Code | Color       |
| ---- | ----------- |
| 1;31 | Bold red    |
| 1;32 | Bold green  |
| 1;33 | Bold yellow |
| 1;34 | Bold blue   |
| 1;35 | Bold purple |
| 1;36 | Bold cyan   |

------

### Permanent change (recommended)

#### Step 1: Generate config (if not exists)

```
dircolors -p > ~/.dircolors
```

#### Step 2: Edit it

```
nano ~/.dircolors
```

Find the line:

```
DIR 01;34
```

Change it, for example:

```
DIR 01;32   # green folders
```