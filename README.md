Shibboleth Ansible Debian 8 playbook
===============================


Playbok Ansible per installare e configurare un setup esemplificativo di ShibbolethIdP 3 e Shibboleth SP 2.
IdP e SP sono in esecuzione sul medesimo server ma su interfacce di rete differenti.
I servizi configurati da questo playbook sono:

- tomcat7
- slapd
- apache2
- mod_shib2 (Service provider)
- shibboleth (Identity provider)
- mysql

Funzionalità:

- Possibilità di configurare il dominio (non più soltanto example.org)
- Configurazione dinamizzata

Todo

- Scelta tra Apache e Nginx/FastCGI come webserver (per sp)
- Scelta tra Tomcat7 e Jetty come contenitore servlet (per idp)
- schema migrations per DB e LDAP
- logrotate setup per le directory di logging

Per utilizzare questo playbook basta installare ansible in ambiente python2

    pip2 install ansible


Comandi di deployment e cleanup
===============================

Creazione delle chiavi firmate:
    
    # installa easy-rsa
    aptitude install easy-rsa
    cp -Rp /usr/share/easy-rsa/ .
    cd easy-rsa

    # personalizziamo attributi, lunghezza DH, percorso di salvataggio delle chiavi
    nano vars

    # activate environment
    source vars

    # purge all the previous keys/dh/crts
    ./clean-all

    # creates: ca.crt	ca.key	index.txt serial
    ./build-ca

    # creates diffie hellman's 
    # il tempo di creazione varia dalla lunghezza del dh, 4096bit prende qualche minuto in più
    ./build-dh

    # then creates client certificates
    ./build-key idp.example.org
    ./build-key sp.example.org

    # adesso nella directory "keys" troviamo le chiavi
    # da spostare in $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio
    # rinominando
    cp keys/ca.crt $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/cacert.pem
    cp keys/ca.key $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/cakey.pem
    cp keys/idp.$nome_dominio.crt $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/$nome_dominio-cert.pem
    cp keys/idp.$nome_dominio.key $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/$nome_dominio-key.pem    
    cp keys/sp.$nome_dominio.key $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/$nome_dominio-key.pem
    cp keys/sp.$nome_dominio.crt $shibboleth-Idp3-ansible/roles/common/files/$nome_dominio/$nome_dominio-cert.pem


Edita le variabili nel playbook e il file hosts prima di fare l'esecuzione
    
    ansible-playbook playbook.yml -i hosts -v

Se cambi parametri puoi fare un cleanup (sconsigliato in ambienti di produzione perchè disinstalla i software)

    ansible-playbook playbook.yml -i hosts -v --limit idp -e '{ cleanup: true }'

Ricorda di aggiungere gli hostname di idp e sp nel tuo /etc/hosts

Risultato
========================


![Alt text](images/1.png)

![Alt text](images/2.png)

![Alt text](images/3.png)


Note
========================

Crea nel tuo DNS o in /etc/hosts gli hostname idp ed sp se sei in testunical

    10.0.3.22  idp.testunical.it
    10.0.4.22  sp.testunical.it

Gli utenti creati in slapd sono definiti in
    
    roles/slapd/templates/direcory-content.ldif

E' necessario configurare gli hostname in /etc/hosts o utilizzare un nameserver dedicato per accedere al servizio HTTPS
    
    https://sp.testunical.it

Ringraziamenti
========================

Ispirato da Garr Netvolution 2017 (http://eventi.garr.it/it/ws17) e basato sul lavoro di Davide Vaghetti https://github.com/daserzw/IdP3-ansible.


Troubleshooting
========================

slapd: (error:80)

    restart slapd in debug mode
    slapd -h ldapi:/// -u openldap -g openldap -d 65 -F /etc/ldap/slapd.d/ -d 65    
    controllare che i file pem non siano vuoti e che i permessi di lettura consentano openldap+r
    per forzare in caso di certificati problematici utilizzare "directory-config_nocert.ldif"
    la connessione tra shibboleth e idp avviene in locale, in questo setup


slapd: ldap_modify: No such object (32): 

    probabilmente stai tentando di modificare qualcosa che non esiste
    Probabilmente il tipo di database se hdb, mdb o altro (dpkg-reconfigure slapd per modificarlo).
    Verificare la corrispondenza tra la configurazione di slapd e il file directory-config

"Request failed: <urlopen error ('_ssl.c:565: The handshake operation timed out',)>"

    TASK [mod-shib2 : Add IdP Metadata to Shibboleth SP]
    libapache2-mod-shib2 non contiene i file di configurazione in /etc/shibboleth (stranezza apparsa su una jessie 8.0 aggiornata a 8.7). 
    Verificare la presenza di questi altrimenti ripopolare la directory

Test confgurazioni singoli servizi/demoni

    apache2ctl configtest
    shibd -t
    

Altri comandi
========================

Esecuzione di un solo role
    
    ansible-playbook playbook.yml -i hosts --tag common

Esecuzione selettiva, quei roles limitati ai nodi idp
    
    ansible-playbook playbook.yml -i hosts -v --tag tomcat7,slapd --limit idp
    
Esecuzione selettiva, quei roles limitati a quel target

    ansible-playbook playbook.yml -i hosts -v --tag tomcat7,slapd --extra-vars "target=idp"

Setup di Shibboleth Idp3
    
    ansible-playbook playbook.yml -i hosts -v --tag shib3idp --limit idp 


    
