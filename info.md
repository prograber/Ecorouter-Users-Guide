Таблица IP-адресов и интерфейсов
| Устройство | Интерфейс | IP-адрес       | Назначение         |
| ---------- | --------- | -------------- | ------------------ |
| ISP        | ens3      | DHCP/WAN       | Интернет           |
| ISP        | ens4      | 172.16.1.1/28  | Сеть HQ            |
| ISP        | ens5      | 172.16.2.1/28  | Сеть BR            |
| HQ-RTR     | isp       | 172.16.1.2/28  | Подключение к ISP  |
| HQ-RTR     | vl100     | 10.10.100.1/27 | HQ-SRV VLAN100     |
| HQ-RTR     | vl200     | 10.10.200.1/27 | HQ-CLI VLAN200     |
| HQ-RTR     | vl999     | 10.10.30.1/29  | Management VLAN999 |
| HQ-RTR     | tunnel.0  | 10.10.10.1/30  | GRE Tunnel         |
| HQ-SRV     | ens3      | 10.10.100.2/27 | DNS/BIND9          |
| HQ-CLI     | ens3      | DHCP           | Клиент VLAN200     |
| BR-RTR     | isp       | 172.16.2.2/28  | Подключение к ISP  |
| BR-RTR     | br        | 10.20.20.1/28  | LAN BR             |
| BR-RTR     | tunnel.0  | 10.10.10.2/30  | GRE Tunnel         |
| BR-SRV     | ens3      | 10.20.20.2/28  | Сервер BR          |

VLAN-сети
| VLAN    | Подсеть        | Шлюз        |
| ------- | -------------- | ----------- |
| VLAN100 | 10.10.100.0/27 | 10.10.100.1 |
| VLAN200 | 10.10.200.0/27 | 10.10.200.1 |
| VLAN999 | 10.10.30.0/29  | 10.10.30.1  |

GRE Tunnel
| Устройство | Tunnel IP     |
| ---------- | ------------- |
| HQ-RTR     | 10.10.10.1/30 |
| BR-RTR     | 10.10.10.2/30 |

peplixmain
prograber
nezlidna


[159](https://drive.google.com/file/d/1EWXvP8OvKOvKSGRwPLM2TF6_zev-5R3_/view?usp=sharing)
[168](https://drive.google.com/file/d/18RF5R4BkNdHLRalFe4PldWgRUtgNR_34/view?usp=drive_link)
[52](https://drive.google.com/file/d/1UqwrHs-qSLwcZaTSgXY44yDJjRWT0ZmF/view?usp=sharing)
