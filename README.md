# Log parser via crontab

```bash
#!/bin/bash

DBA1=admin1@hostname.localdomain
DBA2=admin2@hostname.localdomain
DBA3=admin3@hostname.localdomain
DBA4=admin4@hostname.localdomain

EMAILS="$DBA1,$DBA2,$DBA3,$DBA4"

notify() {
  [ -n "$3" ] && SUBJECT="$3" || SUBJECT="Crontab Script Error"
  echo -e "$1" | mailx -s "$SUBJECT" "$2"
  exit
}

SCRIPT_NAME=$0
SCRIPT_PATH=`readlink -f $0`
SCRIPT_DIR=`dirname $SCRIPT_PATH`

[ ! -d "$SCRIPT_DIR" ] && notify "dir '$SCRIPT_DIR' not exists.." "$EMAILS"

FILT=$SCRIPT_DIR/.$SCRIPT_NAME.filtered
COMP=$SCRIPT_DIR/.$SCRIPT_NAME.compared

[ -d "$FILT" ] && notify "'$FILT' cannot be dir.." "$EMAILS"
[ -d "$COMP" ] && notify "'$COMP' cannot be dir.." "$EMAILS"

touch $FILT $COMP 2>/dev/null

[ "$?" != "0" ] && notify "permission denied?\n - $FILT\n - $COMP" "$EMAILS"

KEYWORDS_1="err|crit|fail|warn|alert|emerg|denied|deny"
KEYWORDS_2="unread|unreachable|missing|problem" # reject

KEYWORDS="$KEYWORDS_1|$KEYWORDS_2"

# BEGIN ######################

VARLOG_DIR=/var/log

VARLOG_KW_KERN="$KEYWORDS"
VARLOG_KW_BOOT="$KEYWORDS"
VARLOG_KW_AUTH="password check failed|authentication failure|$KEYWORDS"
VARLOG_KW_DPKG="upgrade|install|purge|remove|$KEYWORDS"

tail -25000 $VARLOG_DIR/kern.log 2>/dev/null | grep -iE "$VARLOG_KW_KERN" >> $FILT
tail -25000 $VARLOG_DIR/boot.log 2>/dev/null | grep -iE "$VARLOG_KW_BOOT" >> $FILT
tail -25000 $VARLOG_DIR/secure 2>/dev/null | grep -iE "$VARLOG_KW_AUTH" >> $FILT
tail -25000 $VARLOG_DIR/yum.log 2>/dev/null | grep -iE "$VARLOG_KW_DPKG" >> $FILT

# END ######################

RES=`diff $FILT $COMP`

cat $FILT > $COMP

rm -f $FILT

[ -n "$RES" ] && notify "$RES" $EMAILS "Error from log"
```
