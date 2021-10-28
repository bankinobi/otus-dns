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

Split-dns настраивается с помощью списка доступа клиентов `acl` и раздельных `view` под каждого клиента

Создаем два `acl`

```
acl client1 { 192.168.50.15; };
acl client2 { 192.168.50.16; };
```

И отдельные `view`

<details>
<summary>Client1</summary>

    view "client1" {
        match-clients { client1; };
        recursion yes;
        allow-query { client1; };

        // dns.lab zone
        zone "dns.lab" {
            type master;
            allow-transfer { key "zonetransfer.key"; };
            file "/etc/named/named.client1.dns.lab";
        };

        // dns.lab zone reverse
        zone "50.168.192.in-addr.arpa" {
            type master;
            allow-transfer { key "zonetransfer.key"; };
            file "/etc/named/named.client1.dns.lab.rev";
        };

        // newdns.lab zone
        zone "newdns.lab" {
            type master;
            allow-transfer { key "zonetransfer.key"; };
            file "/etc/named/named.newdns.lab";
        };
    };
</details>


<details>
<summary>Client2</summary>

    view "client2" {
        match-clients { client2; };
        recursion yes;
        allow-query { client2; };

        // dns.lab zone
        zone "dns.lab" {
            type master;
            allow-transfer { key "zonetransfer.key"; };
            file "/etc/named/named.dns.lab";
        };

        // dns.lab zone reverse
        zone "50.168.192.in-addr.arpa" {
            type master;
            allow-transfer { key "zonetransfer.key"; };
            file "/etc/named/named.dns.lab.rev";
        };

    };
</details>

Для `client1` создан отдельный файл зоны `named.client1.dns.lab`

Таким образом, client1 видет зоны `dns.lab` и `newdns.lab`. Но в первой зоне ему будет видна только запись `web1`.

Clien2 видит только зону `dns.lab`

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
