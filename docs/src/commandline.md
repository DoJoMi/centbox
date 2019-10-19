## Command-Line Usage/Regex

------------------------------------------------------------------------

```shell
# command
c                                     
# write in foo                  
c > foo                              
# read from foo                  
c < foo      
# append to foo                                        
c >> foo      

# pipe to foo                                       
c | foo                  
# execute 14 from history  
!14                 
# shell command usage
$(c)
# matching a pattern in foo 
grep foo
# egrep can find between | symbol
ls | egrep "mp4|avi"
# show direct log f.e usb
tail -f /var/log/secure

# find all in / with name foo 
find / -name '*.foo'
# find all in / with name foo and size > 100MB
find / -name '*.foo' | -size +100M
# find all in / with name foo and size > 100MB and remove
find / -name '*.foo' | -size +100M -exec /bin/rm {} \;
find / -name '*.foo' | -size +100M | xargs rm
# find all in / with name foo and size and change *foo to -rwx-r-x-rx
find / -name '*.foo' | -size +100M -exec chmod 755 {}\;

# make more directories
mkdir /foo/{}
# cp all dir under foo to actual dir
cp -r /foo .
# list, sort by filesize and append to foo 
ls | sort -n >> foo
# show foo
cat foo 
# write until eof into foo 
cat > foo <<eof foo eof
# cp foo with timestamp
cp -p foo foo.$(date +%F)

# STDERR to /dev/null 
2>/dev/null 
# STDERR to foo
2> foo 
# STDERR & STDOUT to foo
foo 2>&1

# show process foo
ps aux | grep foo
# close process 
pidof | xargs kill -9
# destroy bar 
pkill bar
# destroy all process of solo
kill -9 $(lsof -t -u solo)
# destroy all process of solo
pkill -KILL -u solo 
# show info about running services
chkconfig --list
# status info of foo
systemctl status foo

# replace foo with bar in foo
sed 's/foo/bar/g' foo
# line 3 of foo 
sed -e '1,2d' foo

# cut field 1&7 of foo
cut -d: -f1,7 foo
# cut field of uname
uname -a | cut -d" " -f1

# show the 1&2 columns of foo 
awk '{print $1,$2}' foo 
# show $5 field of foo  
awk -F: '{print $5}' foo
```

### File-Permissions

```shell
suid owner group other
0    7      5       0

           2
           |
   4 <--- ||| ---> 1
          111  <------------ Binary Number 111 (3 digits)
Read <--- ||| ---> Execute
           |
         Write


Permission     Binary  Decimal
----------     ------  -------
Read/Write      110       6
Read Only       100       4
Read/Execute    101       5
```

### Regex

```shell
|:-------------------|---------------------:|
|         ^          |   start of string    |
|         $          |   end of string      |
|         *          |   any number         |
|         .          |   any char exp.line  |
|         []         |   one inside         |
|         [^]        |   no one inside      |
|         {n}        |   exactly n-times    |
|         {n,m}      |   n to m-times       |
|         {n,}       |   n or more          |
|         +          |   one or more        |
|         \          |   escape             |
|         ?          |   once or more       |
|        \d          |   one digit          |
|        \w          |   one word_char      |
|        \s          |   white space char   |
|        \D          |   is not digit       |
|:-------------------|---------------------:| ... many others

/^[a-zA-Z0-9]{6,18}$/
||     |       |   |
|-------------------------------- delimeter   
 |     |       |   |
 |------------------------------- start of string
       |       |   |
       |------------------------- one of this char,int...
               |   |
               |----------------- from 6-18
                   |
                   |------------- end of sting
```