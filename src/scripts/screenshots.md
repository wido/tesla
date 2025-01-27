# Local Screenshots

Tesla has a screenshot function you can call whenever you want

## CID

```console
tesla1@cid-RedactedVIN$  curl -s http://cid:4070/screenshot
```

## IC

To get screenshots from the IC display, from the CID

```console
tesla1@cid-RedactedVIN$ curl -s http://ic:4130/screenshot
tesla1@cid-RedactedVIN$ scp -rp root@ic:/home/tesla/.Tesla/data/screenshots/ /home/tesla/.Tesla/data/
```

# Save and view screenshots

## Option A) Upload images to imgur.com

Save upload-image.sh

```bash
#!/bin/bash

imgurAPI="/var/root/lunars/src/scripts/imgur.sh"
if [ ! -f "$imgurAPI" ]; then
    echo "Downloading imgur library"
    curl https://raw.githubusercontent.com/tremby/imgur.sh/master/imgur.sh -o $imgurAPI
fi

get_path_from_screenshot() {
    echo -e $1 | sed -e "s/\"//g;s/\\\//g;s/_rval_ : //g;s/--/NaN/g;s/ //1" | sed -e 's/[{}]//g'
}

bklght=$(lv GUI_backlightUserRequest)
sdv GUI_backlightUserRequest 100
CID=$(curl -s http://cid:4070/screenshot)
IC=$(curl -s http://ic:4130/screenshot)
CIDPATH=$(get_path_from_screenshot "$CID")
ICPATH=$(get_path_from_screenshot "$IC")
scp -rp root@ic:"$ICPATH" /home/tesla/.Tesla/data/screenshots/
sdv GUI_backlightUserRequest $bklght
bash $imgurAPI $CIDPATH $ICPATH

```

## Option B) Email as attachment

```bash
#!/bin/bash

get_path_from_screenshot() {
        echo -e $1 | sed -e "s/\"//g;s/\\\//g;s/_rval_ : //g;s/--/NaN/g;s/ //1" | sed -e 's/[{}]//g'
}

#SMTP for sending email
server="smtps://smtp.somwhere.com:465"
m_from="yoursender@somwhere.com"
m_to="yourdestination@somwhereelse.com"
m_usr="yoursender"
m_pwd="yoursender_password"
m_file="/tmp/imgmail.html"
m_data="/tmp/imgmail.txt"

# Save the backlight to a variable, set it to 100, then set it back after imgur.sh ?
bklght=$(lv GUI_backlightUserRequest)
sdv GUI_backlightUserRequest 100
CID=$(curl -s http://cid:4070/screenshot)
IC=$(curl -s http://ic:4130/screenshot)
CIDPATH=$(get_path_from_screenshot "$CID")
ICPATH=$(get_path_from_screenshot "$IC")
scp -rp root@ic:"$ICPATH" /home/tesla/.Tesla/data/screenshots/

echo "<html>
<body>
    <div>
        <p>Hello Master, </p>
        <p>Please see the attached screen shots:</p>
        <p>CID</p>
        <img src=\"cid:png_cid.png\"width=\"150\" >
        <p>IC</p>
        <img src=\"cid:png_ic.png\" width=\"150\" >
    </div>
</body>
</html>" > $m_file

mail_from="Your Tesla <$m_from>"
mail_to="The Master <$m_to>"
mail_subject="Requested Screenshots"
mail_reply_to="Your Tesla <$m_from>"
mail_cc=""

function add_file {
    echo "--MULTIPART-MIXED-BOUNDARY
Content-Type: $1
Content-Transfer-Encoding: base64" >> "$m_data"

    if [ ! -z "$2" ]; then
        echo "Content-Disposition: inline
Content-Id: <$2>" >> "$m_data"
    else
        echo "Content-Disposition: attachment; filename=$4" >> "$m_data"
    fi
    echo "$3

" >> "$m_data"
}

message_base64=$(cat $m_file | base64)

echo "From: $mail_from
To: $mail_to
Subject: $mail_subject
Reply-To: $mail_reply_to
Cc: $mail_cc
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary=\"MULTIPART-MIXED-BOUNDARY\"

--MULTIPART-MIXED-BOUNDARY
Content-Type: multipart/alternative; boundary=\"MULTIPART-ALTERNATIVE-BOUNDARY\"

--MULTIPART-ALTERNATIVE-BOUNDARY
Content-Type: text/html; charset=utf-8
Content-Transfer-Encoding: base64
Content-Disposition: inline

$message_base64
--MULTIPART-ALTERNATIVE-BOUNDARY--" > "$m_data"

image_base64=$(cat $CIDPATH | base64)
add_file "image/png" "png_cid.png" "$image_base64"
image_base64=$(cat $ICPATH | base64)
add_file "image/png" "png_ic.png" "$image_base64"

echo "--MULTIPART-MIXED-BOUNDARY--" >> "$m_data"

curl -u $m_usr:$m_pwd -n --ssl-reqd --mail-from "<$m_from>" --mail-rcpt "<$m_to>" --url $server -T $m_data

rm $m_file
rm $m_data
rm $CIDPATH
rm $ICPATH
sleep 3
sdv GUI_backlightUserRequest $bklght

```
