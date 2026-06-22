wtf is this jankyass parsing 🥀🥀🥀  
```
passwd_org=$(ubus call ssh get 2>/dev/null | grep 'Password": ' | awk -F': ' '{print $2}')
passwd_left=$(echo ${passwd_org#*\"})
passwd=$(echo ${passwd_left%%\"*})
ubus call ssh set "{'WANState': '$wanssh_enable', 'LANState': '$lanssh_enable', 'Password':'$passwd'}"
```

oh and heres how root password is set:
(tl;dr: same as ssh/dropbear hash)  
at /rom/etc/init.d/authsystem:  
```
ubus call ssh set "{'WANState': '$wanssh_enable', 'LANState': '$lanssh_enable', 'Password':'$passwd'}"
```
i mean it basically just set it again, worth noting tho
then at /usr/libexec/rpcd/ssh:  
```
echo -e "$Password\n$Password" | passwd root > /tmp/sshpaswd.log
```
which get it from uci get dropbear.@dropbear[0].password  
then at /usr/bin/update_lan_ssh_passkey.sh:  
```
#!/bin/sh

MAC=$(fw_printenv | grep eth1addr | cut -d= -f2 | tr -d ':')
COMBINED="${MAC}FTEL"
MD5_VALUE=$(echo -n "$COMBINED" | md5sum | cut -d' ' -f1)

logger -t "MAC_MD5" "Original MAC: $(fw_printenv | grep eth1addr)"
logger -t "MAC_MD5" "Combined string: $COMBINED"
logger -t "MAC_MD5" "MD5 hash: $MD5_VALUE"

uci set dropbear.@dropbear[0].password=$MD5_VALUE
uci commit dropbear

exit
```
thats it, password is md5((fw_printenv("eth1addr").replace(":", "") + "FTEL")  
and yeah, thats a hash  
do note that fsr my 5.whatever firmware uses AeiAdminPwd while the newer 10.alsowhatever use whatever the thing i just said  
this is probably because of the ssh check in /rom/etc/init.d/authsystem:  
```
    if [ "${AEI_SSH_ENABLED}" == "0" ]; then
        AEI_ADMIN_PWD=`fw_printenv  | grep AeiAdminPwd | cut -d '=' -f 2`
        echo -e "authsystem:${AEI_ADMIN_PWD}" >> /tmp/passlog.txt 2>&1
        echo -e "${AEI_ADMIN_PWD}\n${AEI_ADMIN_PWD}" | passwd root >> /tmp/passlog.txt 2>&1
    else
        lanssh_enable=$(ubus call ssh get 2>/dev/null | grep 'LANState": "1"' -c)
        wanssh_enable=$(ubus call ssh get 2>/dev/null | grep 'WANState": "1"' -c)
        passwd_org=$(ubus call ssh get 2>/dev/null | grep 'Password": ' | awk -F': ' '{print $2}')
        passwd_left=$(echo ${passwd_org#*\"})
        passwd=$(echo ${passwd_left%%\"*})
        ubus call ssh set "{'WANState': '$wanssh_enable', 'LANState': '$lanssh_enable', 'Password':'$passwd'}"
    fi
```
if AEI_SSH_ENABLED is "false"(wtf is "0") then it use AeiAdminPwd, if not then from the above  
