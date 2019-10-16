# Multiplexer

------------------------------------------------------------------------

## Tmux

**For more configuration details** <https://github.com/DoJoMi/dottmux>

## Screen

```shell
yum -y install screen

|:-------------------|--------------------------------------:|
|    crtl + A + C    |     create                            |
|    crtl + A + N    |     switch next                       |
|    crtl + A + P    |     switch previos                    |
|    crtl + A + 0-9  |     switch term 0-9                   |
|    crtl + A + D    |     pause                             |
|    exit            |     delete                            |
|:-------------------|--------------------------------------:|


|:-------------------|--------------------------------------:|
|    screen -x       |     share session with the same user  |
|:-------------------|--------------------------------------:|
```

```shell
# restore lost SSH Connection
screen -R

# there are several suitable screens on:
2123.pts …
3332.pts … 

# then just type to restore
screen -R 2123
```

share session between users

```shell
# bob
screen
crtl + A 
multiuser on
addacl alice

# alice
screen -x bob/
```
