

# Notes/Tips/Resources/Q&A for ECE695 HSS Lab 3

## host cannot ssh to armory

**`After you have successfully flashed GoTEE to the armory.`**

GoTEE sets its ip to 10.0.0.1 by default. [source code](https://github.com/usbarmory/GoTEE-example/blob/89befc1b66c4b6ec2c870ed38672b19fa1732f04/trusted_os_usbarmory/main.go#L33)

```bash
ssh gotee@10.0.0.1
```

If ssh failed, in my case it is because the ip of the armory conflicts with the ip of my router.

```bash
$ ip route get 10.0.0.1  
10.0.0.1 dev wlp6s0 src 10.0.0.56 uid 1000 
```

Find the interface name for armory. Usually it is `enx1a5589a26942` or `usb0`. yours can be different.

unplug and replug the armory

```bash
dmesg | grep -i eth
# and I see kernel message like this:

[95350.828215] usb 1-6: Product: RNDIS/Ethernet Gadget
[95350.843027] cdc_ether: probe of 1-6:1.0 failed with error -16
[95350.843050] usbcore: registered new interface driver cdc_ether
[97522.387555] usb 1-6: Product: RNDIS/Ethernet Gadget
[97522.390911] cdc_ether 1-6:1.0 usb0: register 'cdc_ether' at usb-0000:00:14.0-6, CDC Ethernet Device, 1a:55:89:a2:69:42
[97522.395020] cdc_ether 1-6:1.0 enx1a5589a26942: renamed from usb0
```

```bash
# pick the USB NIC
IF=enx1a5589a26942

# clean slate + set your host IP
sudo ip addr flush dev $IF
sudo ip addr add 10.0.0.2/24 dev $IF
sudo ip link set $IF up

ssh-keygen -f "/home/your_username/.ssh/known_hosts" -R "10.0.0.1"
ssh gotee@10.0.0.1
```

or you can switch to a different ip address.

Change the ip address of the armory to 10.77.0.2/30.
```bash
# pick the USB NIC
IF=enx1a5589a26942

# clean slate + set your host IP
sudo ip addr flush dev $IF
sudo ip addr add 10.77.0.1/30 dev $IF # assign the 10.77.0.1 to the host so that armoy and host are under the same subnet
sudo ip link set $IF up

ping -c1 10.77.0.2
ssh gotee@10.77.0.2
```


sanity check

```bash
ip -4 addr show dev $IF            # should show 10.77.0.1/30
71: enx1a5589a26942: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 1000
    inet 10.77.0.1/30 scope global enx1a5589a26942
       valid_lft forever preferred_lft foreve
```


## Tamago api

This is the API layer of how Tamago framework interacts with the armory hardware. 

[Tamago ARM API](https://pkg.go.dev/github.com/f-secure-foundry/tamago@v0.0.0-20220307101044-d73fcdd7f11b/arm)

Example:

```go
import (
	"log"
	"github.com/usbarmory/tamago/arm"
)
```

```go
func CacheTimerDemo() {
	cpu := arm.CPU{}
	cpu.EnableSMP()
	cpu.EnableCache()
	cpu.InitGenericTimers(0, 0)
}
```

Howevery, the API is limited. But you can study the assembly code e.g. arm/cache.go and arm/cache.s, arm/timer.go and arm/timer.s to understand how the underlying hardware is interfaced. To understand what register controls what part of the hardware, you have to look up the [ARMv7 manual CP15](https://developer.arm.com/documentation/ddi0360/f/control-coprocessor-cp15?lang=en). With the knowledge of manipulating CP15 registers, you can do interesting things like Invalidate Data TLB Register


## Cache organization

Knowing how physical address is mapped to cache, set associative and replacement policy will be helpful.
[Cache1 Lecture](https://courses.cs.washington.edu/courses/cse410/10sp/lectures/11-cache-1.pdf) from CSE 410 University of Washington.
[Cache2 Lecture](https://courses.cs.washington.edu/courses/cse410/10sp/lectures/12-cache-2.pdf) from CSE 410 University of Washington.