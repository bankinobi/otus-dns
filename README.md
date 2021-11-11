# Otus DNS

```
* взять стенд https://github.com/erlong15/vagrant-bind

* добавить еще один сервер client2

* завести в зоне dns.lab имена

    web1 - смотрит на клиент1

    web2 - смотрит на клиент2

* завести еще одну зону newdns.lab

    завести в ней запись www - смотрит на обоих клиентов

* настроить split-dns

    клиент1 - видит обе зоны, но в зоне dns.lab только web1

    клиент2 - видит только dns.lab

* настроить все без выключения selinux
```

Сервер `client2` добавлен в Vagrantfile

В зоне `dns.lab` добавлены записи
```
; WEB
web1            IN      A       192.168.50.15
web2            IN      A       192.168.50.16
```

Создана дополнительная зона `newdns.lab`, в ней созданы A записи для корректной работы CNAME
```
; WWW
newdns.lab.     IN      A       192.168.50.15
newdns.lab.     IN      A       192.168.50.16
www             IN      CNAME   newdns.lab.
```

Split-dns настраивается с помощью списка доступа клиентов `acl` и раздельных `view` под каждого клиента.

Для корректной работы secondary сервера, нам так же понадобятся два дополнительных ключа:

```
key "client1-key" {
    algorithm hmac-md5;
    secret "PjaK51ukqjgZhkVMyl3N0A==";
};
key "client2-key" {
    algorithm hmac-md5;
    secret "69mwmAvUq99KIxVNCmoy2g==";
};
```


Создаем два `acl`

```
acl client1 { !key client2-key; key client1-key; 192.168.50.15; };
acl client2 { !key client1-key; key client2-key; 192.168.50.16; };
```

И отдельные `view`

<details>
<summary>Client1</summary>

    view "client1" {
        match-clients { client1; };

        // dns.lab zone
        zone "dns.lab" {
            type master;
            file "/etc/named/named.client1.dns.lab";
            also-notify { 192.168.50.11 key client1-key; };
        };

        // dns.lab zone reverse
        zone "50.168.192.in-addr.arpa" {
            type master;
            file "/etc/named/named.client1.dns.lab.rev";
            also-notify { 192.168.50.11 key client1-key; };
        };

        // newdns.lab zone
        zone "newdns.lab" {
            type master;
            file "/etc/named/named.newdns.lab";
            also-notify { 192.168.50.11 key client1-key; };
        };
    };
</details>


<details>
<summary>Client2</summary>

    view "client2" {
        match-clients { client2; };

        // dns.lab zone
        zone "dns.lab" {
            type master;
            file "/etc/named/named.dns.lab";
            also-notify { 192.168.50.11 key client2-key; };
        };

        // dns.lab zone reverse
        zone "50.168.192.in-addr.arpa" {
            type master;
            file "/etc/named/named.dns.lab.rev";
            also-notify { 192.168.50.11 key client2-key; };
        };
    };
</details>

Для `client1` создан отдельный файл зоны `named.client1.dns.lab`

Таким образом, client1 видет зоны `dns.lab` и `newdns.lab`. Но в первой зоне ему будет видна только запись `web1`.

Clien2 видит только зону `dns.lab`

<details>
<summary>Primary server:</summary>

    [vagrant@client1 ~]$ dig any web1.dns.lab

    ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> any web1.dns.lab
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 13818
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;web1.dns.lab.			IN	ANY

    ;; ANSWER SECTION:
    web1.dns.lab.		3600	IN	A	192.168.50.15

    ;; AUTHORITY SECTION:
    dns.lab.		3600	IN	NS	ns02.dns.lab.
    dns.lab.		3600	IN	NS	ns01.dns.lab.

    ;; ADDITIONAL SECTION:
    ns01.dns.lab.		3600	IN	A	192.168.50.10
    ns02.dns.lab.		3600	IN	A	192.168.50.11

    ;; Query time: 0 msec
    ;; SERVER: 192.168.50.10#53(192.168.50.10)
    ;; WHEN: Thu Nov 11 10:15:30 UTC 2021
    ;; MSG SIZE  rcvd: 127


    [vagrant@client1 ~]$ ping web1.dns.lab
    PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.020 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.044 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.050 ms
    [vagrant@client1 ~]$ ping web2.dns.lab
    ping: web2.dns.lab: Name or service not known
    [vagrant@client1 ~]$ ping www.newdns.lab
    PING newdns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.024 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.066 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.048 ms
    [vagrant@client2 ~]$ ping web1.dns.lab
    PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.462 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.520 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.542 ms
    [vagrant@client2 ~]$ ping web2.dns.lab
    PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=1 ttl=64 time=0.016 ms
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=2 ttl=64 time=0.045 ms
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=3 ttl=64 time=0.047 ms
    [vagrant@client2 ~]$ ping www.newdns.lab
    ping: www.newdns.lab: Name or service not known
</details>

<details>
<summary>Secondary server:</summary>

    [vagrant@client1 ~]$ dig any web1.dns.lab

    ; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.7 <<>> any web1.dns.lab
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42397
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;web1.dns.lab.			IN	ANY

    ;; ANSWER SECTION:
    web1.dns.lab.		3600	IN	A	192.168.50.15

    ;; AUTHORITY SECTION:
    dns.lab.		3600	IN	NS	ns02.dns.lab.
    dns.lab.		3600	IN	NS	ns01.dns.lab.

    ;; ADDITIONAL SECTION:
    ns01.dns.lab.		3600	IN	A	192.168.50.10
    ns02.dns.lab.		3600	IN	A	192.168.50.11

    ;; Query time: 1 msec
    ;; SERVER: 192.168.50.11#53(192.168.50.11)
    ;; WHEN: Thu Nov 11 10:16:21 UTC 2021
    ;; MSG SIZE  rcvd: 127
    [vagrant@client1 ~]$ ping web1.dns.lab
    PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.015 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.059 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.061 ms
    [vagrant@client1 ~]$ ping web2.dns.lab
    ping: web2.dns.lab: Name or service not known
    [vagrant@client1 ~]$ ping www.newdns.lab
    PING newdns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.015 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.043 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.051 ms
    [vagrant@client2 ~]$ ping web1.dns.lab
    PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=1 ttl=64 time=0.462 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=2 ttl=64 time=0.520 ms
    64 bytes from web1.dns.lab (192.168.50.15): icmp_seq=3 ttl=64 time=0.542 ms
    [vagrant@client2 ~]$ ping web2.dns.lab
    PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=1 ttl=64 time=0.016 ms
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=2 ttl=64 time=0.045 ms
    64 bytes from web2.dns.lab (192.168.50.16): icmp_seq=3 ttl=64 time=0.047 ms
    [vagrant@client2 ~]$ ping www.newdns.lab
    ping: www.newdns.lab: Name or service not known
</details>


## Настроить все без выключения selinux

Вспоминая ДЗ по Selinux, мы уже знаем в чем проблема.

Поэтому на ns серверах добавляем правильный file context для директории /etc/named
```
- name: set proper setype for /etc/named
  sefcontext:
    target: "/etc/named(/.*)?"
    setype: named_zone_t
    state: present

- name: apply file context to /etc/named
  command: restorecon -R -v /etc/named

- name: make sure selinux enabled
  selinux:
    policy: targeted
    state: enforcing
```
