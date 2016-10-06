##### Fiap - Soluções em Redes para ambientes Linux
profhelder.pereira@fiap.com.br

---
# Multiplas entradas para serviços de e-mail

É comum que se utilize mais de um backend de email para aumentar a disponibilidade de seu serviço, para que isso funcione seu DNS deverá prover algum mecanismo de balanceamento de carga entre todos os apontamentos criados, Por padrão utilizamos a definição de preferência do apontamento do tipo MX, essa configuração está prevista e descrita na [rfc974](https://github.com/2TRCR/DNS/blob/master/rfcs/rfc974.txt);

Basicamente cada entrada do tipo MX corresponde a um nome de domínio com dois pedaços de dados, uma refere-se ao valor de preferência (um 16-bit inteiro sem sinal), e o outro refere-se ao nome de um anfitrião, um domínio ou um endereço referente a um backend de email, O número de preferência é usado para indicar em que ordem o serviço de MTA deve tentar entregar a mensagem para os anfitriões MX, sempre do menor para o maior, ou seja, a menor entrada numerada refere-se ao MX s ser usado primeiro. Várias entradas MXs com a mesma preferências são permitidas e têm a mesma prioridade.

Essa configuração deverá ser executada neste formato:

```sh
mta1	IN	MX	10 mta1.fiap.edu.br
mta2	IN	MX	10 mta2.fiap.edu.br
mta3	IN	MX	20 mta3.fiap.edu.br
mta4	IN	MX	30 mta3.fiap.edu.br
```

Por exemplo, considere os apontamentos de e-mail do yahoo e do google:

```sh
# dig -t MX gmail.com
# dig -t MX yahoo.com
```

Uma abordagem alternativa é definir vários registros A com o mesmo nome do servidor de correio, algo no formato abaixo:


mail	IN	A	192.168.1.7
			192.168.1.8
			192.168.1.9


# Ponteiros do tipo Round Robin

Supondo que você deseja aumentar a disponibilidade em um serviço como uma página de conteúdo, você simplesmente poderia definir vários registros A com o mesmo nome e diferentes endereços IPs como no exemplo acima, esse tipo de entrada recebe o nome de RoundRobin ou entrada do tipo RR;

Em tempo o bind9 também suporta que simplesmente sejam criadas varias entradas com a mesma origem:

```sh
ftp	IN	A	192.168.1.7
ftp	IN	A	192.168.1.8
ftp	IN	A	192.168.1.9
```

> Esse modelo de configuração estabelece multiplos ponteiros porem sem que seja executado qualquer balanceamento de carga entre eles, logo, não se trata de um LoadBalancer em sua proposta.


# Balanceamento de requisições com ponteiros SRV

Conteúdo baseado no artigo publicado em [fogonacaixadagua](http://www.fogonacaixadagua.com.br/2009/09/dns-srv-records-com-bind/), Sim o nome é bizarro mas o artigo é muito bom ;)

Entradas do tipo SRV são utlizadas em serviços de DNS principalmente por aplicações responsáveis por backends de autenticação de usuários como os protocolos LDAP e Kerberos mas podem ser encotradas em outros protocolos como o NTP e XMPP embora com menor frequencia.

Basicamente a estrutura de um entrada do tipo SRV é composta da seguinte forma:

***<< _Serviço._Proto.Name TTL Classe SRV Prioridade Peso Porta Destino >>***

Onde:

- Serviço: nome simbólico para o serviço
- Proto: protocolo do serviço; usualmente é TCP ou UDP.
- Name: o domínio para o qual a entrada é válida
- TTL: Time to live da entrada
- Classe: sempre é IN nesse caso
- Prioridade: a prioridade para o host de destino, valores menores significam maior preferência.
- Peso: um peso relativo à entrada com a mesma prioridade
- Porta: oa porta UDP ou TCP na qual o serviço será encontrado.
- Destino: é a entrada do DNS para o host que está provendo o serviço.

Na pratica teriamos algo mais ou menos assim:

***<<_sip._tcp.fiap.edu.br. 86400 IN SRV 0 5 5060 192.168.1.10 >>***

Onde entra o balanceamento de carga? No fato de que assim como as entradas RR um ponteiros SRV não precisa ser unico, aliases ou CNAMEs não podem ser usados como destinos válidos e neste contexto podemos alterar o valor dos campos Peso e Prioridade para assim criar nosso esquema de balanceamento de carga:

```sh
_sip._tcp.fiap.edu.br. 86400 IN SRV 10 50 5060 192.168.1.10
_sip._tcp.fiap.edu.br. 86400 IN SRV 10 40 5060 192.168.1.11
_sip._tcp.fiap.edu.br. 86400 IN SRV 20 50 5060 192.168.1.12
_sip._tcp.fiap.edu.br. 86400 IN SRV 20 50 5060 192.168.1.12
_sip._tcp.fiap.edu.br. 86400 IN SRV 20 0 5060 192.168.1.13
```

1. No contexto acima adicionei duas entradas com prioridade 10, porém dentre elas uma delas possui peso superior, logo o servidor 192.168.1.10 deverá receber algo em otrno de 60% das requisições enquanto o servidor 192.168.1.11 receberá 40%.

2. Se ambos os servidores ( .10 e .11 ) estiverem indisponíveis os servidor 192.168.1.12 e 192.168.1.13 deverão receber essas requisições com uma balanceameno  igual de carga; 

3. De forma analoga o servidor 192.168.1.14 fará o mesmo caso os 4 servidores anteriores estiverem offline.

> Aqui já podemos falar em balanceamento de carga porém de forma limitada uma vez que a informação é estática, ou seja, não há monitoração responsável por remover um servidor da lista caso ele esteja inascesível, apenas existem ponteiros SRV determinando a prioridade na tentativa de acesso, além disso, a carga dos servidores em questão também não é levado em conta pelo bind9.



