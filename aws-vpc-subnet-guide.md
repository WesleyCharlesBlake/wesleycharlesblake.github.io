## AWS Subnets
VPC Subnetting can be quite tricky to figure out in the beginning. One thing I have noticed, is that its worth the time and effort in getting the planning done first, before you deploy anything.

Ideally we want a VPC, to deploy our instances to, and to have enough IP addresses to cater for our scale out strategy.

We also want to deploy the public facing instances in a separate subnet to private/internal resources. Another best practice would be to have each tier of your architecture on its own subnet (Public/Web, App, DB, Workers etc). This way we can plan a really rock solid infrastructure, that is secure,  and minimise horrible surprises down the line.

### Fault-Tolerance
AWS provides geographic distribution out of the box in the form of Availability Zones (AZs). Every region has at least two.

A Simple reference to help:
```
16-bit: 65534 addresses
18-bit: 16382 addresses
19-bit: 8190 addresses
20-bit: 4094 addresses
```
> be careful not to use networks that are too large. Large broadcasts can affect network performance negatively if you dont know exactly what you are doing.

A simple subnet I use, is cut like this:
```
10.0.0.0/18 — AZ A
    10.0.0.0/19 — Private
    10.0.32.0/19
            10.0.32.0/20 — Public
            10.0.48.0/20 — Spare
```

There is no need to have thousands of public facing addresses in most cases, so I tend to put these instances in a `/20` bit network. I know I have more than enough space to grow. You could cut this down a lot more, again depending on your requirements.

Later on, if you want to add a “Protected” subnet with NACL’s, you just subdivide your Spare space:
```
10.0.0.0/18 — AZ A
      10.0.0.0/19 — Private
      10.0.32.0/19
              10.0.32.0/20 — Public
              10.0.48.0/20
                  10.0.48.0/21 — Protected
                  10.0.56.0/21 — Spare
```
Just make sure whatever you do in one AZ, you duplicate in all the others:

```
10.0.0.0/16:
    10.0.0.0/18 — AZ A
        10.0.0.0/19 — Private
        10.0.32.0/19
               10.0.32.0/20 — Public
               10.0.48.0/20
                   10.0.48.0/21 — Protected
                   10.0.56.0/21 — Spare
    10.0.64.0/18 — AZ B
        10.0.64.0/19 — Private
        10.0.96.0/19
                10.0.96.0/20 — Public
                10.0.112.0/20
                    10.0.112.0/21 — Protected
                    10.0.120.0/21 — Spare
    10.0.128.0/18 — AZ C
        10.0.128.0/19 — Private
        10.0.160.0/19
                10.0.160.0/20 — Public
                10.0.176.0/20
                    10.0.176.0/21 — Protected
                    10.0.184.0/21 — Spare
    10.0.192.0/18 — Spare
```
Your routing tables would look like this:

```
“Public”
    10.0.0.0/16 — Local
    0.0.0.0/0  —  Internet Gateway
```
```
“Internal-only” (ie, Protected and Private)
    10.0.0.0/16 — Local
```

Create those two route-tables and then apply them to the correct subnets in each AZ. You’re done.

Planning really does go along way here, and will save you a lot headaches in the future. Restructuring subnets in production is a hair raising experience! So prevent that from ever happening by being thorough in your planning.
