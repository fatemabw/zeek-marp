---
theme: zeek
_class: lead
---

![bg](https://marp.app/assets/hero-background.svg)

![width:250px](./zeek-logo.png)
# Introduction to Zeek
### A Zeek training
*NSF Cybersecurity Summmit, 2026*

---

# A standard bullet slide

The following items appear bit-by-bit:

* An item with text
* And another
  * with a subitem

---

# Section level 1

## Section level 2

### Section level 3

#### Section level 4

##### Section level 5

---

## A multi section slide

These all appear at once:

- An item with text
- And another

## With another section

- An item with text
- And another

---

# A two-column slide

<div class="columns">
<div>

* An item
* And another

</div>
<div>

* And yet another
* And the last

</div>
</div>

---

# Text on the left, code on the right

<div class="columns">
<div>

This is just text that explains what the thing on the right is all about.

In this case, it's some JSON.

</div>
<div>

<style scoped>
pre { font-size: 65%; }
</style>

```json
{
  "ts": 1117503119.471231,
  "uid": "CuOgIYhboLbVMhTnh",
  "id.orig_h": "192.150.186.238",
  "id.orig_p": 42762,
  "id.resp_h": "66.35.250.209",
  "id.resp_p": 80,
  "proto": "tcp",
  "service": "http",
  "duration": 6.483856916427612,
  "orig_bytes": 377,
  "resp_bytes": 10041,
  "conn_state": "SF",
  "local_orig": false,
  "local_resp": false,
  "missed_bytes": 0,
  "history": "ShADadfF",
  "orig_pkts": 11,
  "orig_ip_bytes": 957,
  "resp_pkts": 10,
  "resp_ip_bytes": 10569,
  "ip_proto": 6
}
```

</div>
</div>

---

# Text on top, code below

For example, for console output:

```console
$ while true; do ls -la | foo; done
bar
```

---

# A big image with some text

![center width:400px](zeek-logo.png)

This is one of several ways to center an image.

---

# Text on the left, image right

<div class="columns">
<div>

This is just text that explains what the thing on the right is all about.

</div>
<div>

![center width:300px](zeek-logo.png)

</div>
</div>

---

<!-- _class: chapter -->
# Moving on!

---

# Let's figure out Zeek

We can mention some inline code, like `&redef`, and we can highlight Zeek script:

```zeek
@load ./foo

event zeek_init() &priority=-5 {
    print "hello world!";
}
```
