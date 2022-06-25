"# temp" 


```bash
find / -name 'tcpdump' -o -name 'gdb' -o -name 'strace'  -o -name 'readelf' -o -name 'cpp' -o -name 'gcc' -o -name 'netcat' -o -name 'nc' -o -name 'nmap'  -o -name 'ethereal' -o -name 'objdump' -o -name 'aplay' -o -name 'arecord' -o -name 'vnstat' -o -name 'vnStatsvg' -o -name 'nload' -o -name 'atop' -o -name 'iftop'

find / -name "*.key" 2>/dev/null |xargs grep "BEGIN RSA" |awk -F ':' '{print $1}'|xargs grep -H -E -o -c "ENCRYPTED" |grep ".key:0"

find / -perm -u=s -type f 2>/dev/null
```
