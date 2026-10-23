# JioRouter AX6000 (OpenWrt): Status LED setup (green / red)

*(English summary neeche hai / English summary below)*

JioRouter AX6000 par teen status LED hoti hain: **blue, green aur red**. Default me **red LED jalti rehti hai**, chahe internet chal raha ho ya nahi. Is guide se aap LED ko kaam ki cheez bana sakte ho.

Teen tareeke hain. Aap **sirf ek** chuno:

| | Kya hota hai | Kahan karna hai | Kab chuno |
|---|---|---|---|
| **Method 1** | Green jalti hai jab **WAN cable (ODU se) juda ho**. Red aur blue band | **Browser me (LuCI)**, SSH nahi | Sabse aasan |
| **Method 2** | Method 1 jaisa hi, par **ek baar paste** karke | **SSH** (PowerShell) | Jab clicks nahi karne |
| **Method 3** | Green = **internet chal raha hai**, Red = **internet band** | **SSH** (PowerShell) | Jab sach me internet-aware LED chahiye |

> **Ek important farak:** Method 1 aur 2 sirf **cable ka link** dekhte hain. ODU band ho jaye ya Jio ka internet chala jaye par cable juda rahe, to green jalti rahegi. **Sirf Method 3 internet ko ping karke dekhta hai.**
>
> **Dono ya teeno ek saath mat chalao.** Wo ek doosre ki LED ko override karte hain.

> **Kya test hua:** LuCI ke dialog ke fields (Name, LED Name, Trigger, Device, Trigger Mode) JIDU6401 par OpenWrt 25.12.6 me dekhe gaye. **Method 2 aur 3 ke scripts router par chala kar abhi confirm nahi hue** hain, to pehle khud test karo. Doosre router par LED ke naam alag ho sakte hain (neeche Step 0 dekho).
>
> Ye ek unofficial community guide hai. Apne risk par use karo, aur pehle config ka backup lo (LuCI → System → Backup / Flash Firmware → Generate archive).

---

## Step 0 (sabke liye): LED ke naam check karo

Aapke router par LED ke naam `blue:status`, `green:status` aur `red:status` hone chahiye. Doosre router par alag ho sakte hain, isliye ek baar check karo.

1. **SSH kholo** (neeche "SSH kaise kholte hain" dekho).
2. Ye chalao:
   ```
   ls /sys/class/leds/
   ```

**Kya dikhna chahiye (JioRouter AX6000):**
```
blue:status  green:status  mt76-phy0  mt76-phy1  red:status
```

`mt76-phy0` aur `mt76-phy1` WiFi ki LED hain, unhe mat chhedna. Naam alag dikhe to guide ke `green:status` wagairah ko apne naam se badal do.

### SSH kaise kholte hain (Windows)
1. **Windows key** dabao, `PowerShell` type karo, **Enter**.
2. Ye likho aur Enter dabao:
   ```
   ssh root@192.168.1.1
   ```
3. Pehli baar "Are you sure..." poochega: `yes` likho.
4. **Router ka password** type karo (type karte waqt kuch dikhta nahi, ye normal hai), Enter.
5. `root@OpenWrt:~#` dikhe to aap router ke andar ho. Commands yahan paste karo (**right-click** se paste hota hai).

> "REMOTE HOST IDENTIFICATION HAS CHANGED" ka error aaye (firmware flash ke baad) to apne PC ke PowerShell me `ssh-keygen -R 192.168.1.1` chalao, phir dobara SSH karo.
>
> Code paste karte waqt **tab nahi, sirf spaces** hone chahiye. Guide ke scripts me spaces hi hain.

---

## Method 1: LuCI se (browser me, SSH nahi)

**Kahan:** router ka panel → `http://192.168.1.1` → **System → LED Configuration**

1. **Add LED action** dabao. Ek dialog khulega jisme ye fields hain: Name, LED Name, Trigger, Interval, aur trigger ke hisaab se Device / Trigger Mode.

2. **Pehli entry (green):**

| Field | Kya bharna hai |
|---|---|
| Name | `Green WAN link` |
| LED Name | `green:status` |
| Trigger | `Network device activity (kernel: netdev)` |
| Interval | **khali chhod do** (grey `50` sirf placeholder hai) |
| Device | `wan` |
| Trigger Mode | `Link On` |

   **Save** dabao.

3. **Doosri entry (red band):** **Add LED action** dobara dabao.

| Field | Kya bharna hai |
|---|---|
| Name | `Red off` |
| LED Name | `red:status` |
| Trigger | `Always off (kernel: none)` (naam thoda alag ho sakta hai, jo "none" ya "off" jaisa ho wahi chuno) |

   **Save** dabao.

4. **Teesri entry (blue band, optional):** wahi, bas `Name` = `Blue off` aur `LED Name` = `blue:status`.

5. Page ke neeche **Save & Apply** dabao.

> **Interval kya hai?** Ye sirf blink wale triggers ke liye hai (LED kitne millisecond jale aur bujhe). **Link On** me LED steady jalti hai, to Interval ka koi asar nahi padta. Traffic par blink chahiye to **Trigger Mode me Transmit aur Receive** bhi tick karo, tab Interval kaam aata hai (`50` tez, `100-200` dheere).

**Kya dikhna chahiye:** green LED jalti hai (ODU juda ho to), red aur blue band. Na ho to router ko ek baar reboot karke dekho.

---

## Method 2: SSH se (Method 1 ka hi kaam, ek baar paste)

**Kahan:** SSH window (`root@OpenWrt:~#`). Step 0 me bataya tareeka.

Ye poora ek baar me paste karo:

```
if [ -x /etc/init.d/net-led ]; then
    /etc/init.d/net-led stop
    /etc/init.d/net-led disable
fi

while uci -q delete system.@led[0]; do :; done

uci set system.led_green_wan='led'
uci set system.led_green_wan.name='Green WAN link'
uci set system.led_green_wan.sysfs='green:status'
uci set system.led_green_wan.trigger='netdev'
uci set system.led_green_wan.dev='wan'
uci set system.led_green_wan.mode='link'

uci set system.led_red_off='led'
uci set system.led_red_off.name='Red off'
uci set system.led_red_off.sysfs='red:status'
uci set system.led_red_off.trigger='none'
uci set system.led_red_off.default='0'

uci set system.led_blue_off='led'
uci set system.led_blue_off.name='Blue off'
uci set system.led_blue_off.sysfs='blue:status'
uci set system.led_blue_off.trigger='none'
uci set system.led_blue_off.default='0'

uci commit system
/etc/init.d/led restart
```

**Ye kya karta hai:**
- Agar Method 3 wala `net-led` pehle se chal raha ho to use **stop aur disable** kar deta hai.
- **Purane saare LED entries hata deta hai** (jo pehle GUI ya command se bane the) aur naye bana deta hai. Dobara chalane par duplicate nahi banenge.

**Kya dikhna chahiye:** wahi jo Method 1 me, green jalti hai aur red/blue band.

Ye wahi cheezein hain jo LuCI me **System → LED Configuration** me dikh sakti hain.

---

## Method 3: Internet-aware (green = internet OK, red = internet band)

**Kahan:** SSH window.

**Kaise kaam karta hai:** router har 15 second me `1.1.1.1` aur `8.8.8.8` ko ping karta hai. Dono me se ek bhi jawab de to internet theek. **2 baar lagataar fail** hone par red ho jata hai (taaki ek chhoti glitch par LED na badle), yaani internet jaane ke lagbhag **30-45 second** baad red. Wapas aane par 15 second me green.

> Is method me LuCI wale LED entries (Method 1/2) nahi hone chahiye, nahi to takrayenge. Pehle `uci show system | grep -i led` chalao. Kuch dikhe to LuCI me **System → LED Configuration** se wo entries hata do.

Ye poora ek baar me paste karo:

```
cat > /usr/bin/net-led.sh <<'EOF'
#!/bin/sh
GREEN=/sys/class/leds/green:status
RED=/sys/class/leds/red:status
BLUE=/sys/class/leds/blue:status
TARGETS="1.1.1.1 8.8.8.8"
INTERVAL=15
FAILS_NEEDED=2

led_set() {
    [ -d "$1" ] || return 0
    echo none > "$1/trigger" 2>/dev/null
    if [ "$2" = 1 ]; then
        cat "$1/max_brightness" > "$1/brightness"
    else
        echo 0 > "$1/brightness"
    fi
}

internet_ok() {
    for t in $TARGETS; do
        ping -c 1 -W 2 "$t" >/dev/null 2>&1 && return 0
    done
    return 1
}

fails=0
state=""
led_set "$BLUE" 0

while true; do
    if internet_ok; then
        fails=0
        if [ "$state" != "ok" ]; then
            led_set "$RED" 0
            led_set "$GREEN" 1
            state="ok"
        fi
    else
        fails=$((fails + 1))
        if [ "$fails" -ge "$FAILS_NEEDED" ] && [ "$state" != "down" ]; then
            led_set "$GREEN" 0
            led_set "$RED" 1
            state="down"
        fi
    fi
    sleep "$INTERVAL"
done
EOF
chmod +x /usr/bin/net-led.sh

cat > /etc/init.d/net-led <<'EOF'
#!/bin/sh /etc/rc.common
START=99
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/net-led.sh
    procd_set_param respawn
    procd_close_instance
}
EOF
chmod +x /etc/init.d/net-led

/etc/init.d/net-led enable
/etc/init.d/net-led start
```

**Test karo:**
1. `ps | grep net-led` chalao. Process dikhna chahiye.
2. 15-20 second me **green** jalni chahiye.
3. Jio ODU ka cable nikaal do (ya ODU band kar do). **30-45 second me red** aa jana chahiye. Cable wapas lagao to green.

> Agar aap Tailscale se bahar se router chala rahe ho to `ifdown wan` se test mat karo, connection toot jayega.

**Customize:** `/usr/bin/net-led.sh` ke upar ke variables: `TARGETS` (ping ke addresses), `INTERVAL` (kitni der baad check), `FAILS_NEEDED` (kitni baar fail hone par red). Rang badalna ho to `GREEN=` ya `RED=` me doosre LED ka path likh do.

---

## Firmware flash ke baad kya bachta hai

- **Method 1 aur 2:** LED settings `/etc/config/system` me hoti hain, jo **"Keep settings"** se aam taur par bachi rehti hain.
- **Method 3:** `/etc/init.d/net-led` aur `/usr/bin/net-led.sh` aapki banayi files hain, jo "Keep settings" nahi bachata. Naya firmware flash karne par dobara install karna pad sakta hai. Bachane ke liye `/etc/sysupgrade.conf` me ye do lines daalte hain (**ye tareeka abhi test nahi hua**):
  ```
  echo "/etc/init.d/net-led" >> /etc/sysupgrade.conf
  echo "/usr/bin/net-led.sh" >> /etc/sysupgrade.conf
  ```
  Flash ke baad LED na badle to `/etc/init.d/net-led enable` aur `start` dobara chalao.

---

## Dikkat aaye to

| Dikkat | Kya karo |
|---|---|
| Red LED abhi bhi jal rahi hai | Method 1 me "Red off" entry **Save & Apply** hui? Router reboot karke dekho |
| Green nahi jalti (Method 1/2) | **Device** me `wan` chuna hai? WAN cable juda hai? `ls /sys/class/leds/` se LED ka naam check karo |
| Green hamesha jalti hai chahe internet band ho | Ye Method 1/2 ka normal hai (cable link dekhta hai). Internet-aware chahiye to Method 3 |
| Method 3 me LED nahi badalti | `ps \| grep net-led` se process dekho. LuCI ke LED entries hata do. `/etc/init.d/net-led restart` chalao |
| Dono LED ajeeb jal rahi hain | Method 1/2 aur Method 3 ek saath chal rahe hain. Ek hi rakho |
| `ls /sys/class/leds/` me naam alag hain | Guide me `green:status`, `red:status` ko apne naam se badal do |
| Tab daalne se script toot gayi | Tab ki jagah spaces use karo |

---

## Hatana ho to

**Method 1/2:**
```
uci delete system.led_green_wan
uci delete system.led_red_off
uci delete system.led_blue_off
uci commit system
/etc/init.d/led restart
```
(Ya LuCI me **System → LED Configuration** se entries delete karke **Save & Apply**.) Reboot ke baad LED purane haal me (red jalti) aa jayegi.

**Method 3:**
```
/etc/init.d/net-led stop
/etc/init.d/net-led disable
rm -f /etc/init.d/net-led /usr/bin/net-led.sh
reboot
```

---

## Surakshit rehne ke liye

- Ye scripts sirf router ki **LED ke files** (`/sys/class/leds/...`) me likhte hain, aur Method 3 me **ping** karte hain. Network ya firewall ki settings nahi badalte.
- **Anjaan jagah se mile scripts bina padhe router par mat chalao.** Is guide ke scripts chhote hain, aap khud padh sakte ho.
- Router ka **password set** rakho. Bina password ke LAN par koi bhi router badal sakta hai.
- Config backup (`.tar.gz`) **private** rakho, GitHub ya kisi public jagah par upload mat karo.

---

## English summary

**What this is:** how to set up the status LEDs on the **JioRouter AX6000** under OpenWrt (tested layout: JIDU6401, OpenWrt 25.12.6). By default the red LED stays on.

**Three methods, pick one:**
1. **LuCI (System, LED Configuration):** green LED follows the WAN link (`netdev` trigger, device `wan`, mode *Link On*), red and blue off. Leave *Interval* empty.
2. **SSH `uci` script:** the same result as method 1 in one paste.
3. **Internet-aware service:** a small procd service pings `1.1.1.1` and `8.8.8.8` every 15 seconds; green when reachable, red after two failed checks.

Methods 1 and 2 only watch the **cable link**, not real internet. Do not combine methods. Scripts for methods 2 and 3 were written from the router's LED layout and are **not yet confirmed on hardware**, so test them before relying on them. Unofficial community guide, use at your own risk.

---

## Credits

OpenWrt ke developers ko dhanyavaad. Ye guide ek real JioRouter AX6000 ke LED layout (`blue:status`, `green:status`, `red:status`) par based hai.
