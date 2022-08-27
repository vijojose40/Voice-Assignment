# Voice-Assignment

STEP 1.Installation of asterisk.

    1a.Make an asterisk listen on the 5080 sip signaling port for the signaling.
    
    cd /etc/asterisk
    vim sip.conf
    set bindport=5080
    and save the file and reload asterisk -rx "sip reload"
    
    
    1b).Make an asterisk to use 10000 to 11000 for the rtp. 
    cd /etc/asterisk
    vim rtp.conf
    [general]
    rtpstart=10000
    rtpend=11000

    save and reload rtp module asterisk -rx "module reload res_rtp_asterisk.so"

    1c.Register two clients from two users 101 and 102 using zoiper/linphone/etc
    
    cd /etc/asterisk
    
   vim sip.conf and below line
   
        [101]
        type=friend
        username=101
        host=dynamic
        secret=pass#101 ;;use a strong password for productions
        context=Testcall 
        transport=udp
        disallow=all
        allow=alaw
        qualify=yes
        nat=rtp_force,comedia



        [102]
        type=friend
        username=102
        host=dynamic
        secret=pass#102 ;;use a strong password for productions
        context=Testcall 
        transport=udp
        disallow=all
        allow=alaw
        qualify=yes
        nat=rtp_force,comedia
        
        save the file and reload asterisk -rx "sip reload"
        
        asterisk -rx "sip show peers" to checking the sip peer
          Name/username             Host                                    Dyn Forcerport Comedia    ACL Port     Status      Description                      
          101/101                   (Unspecified)                            D  No         Yes            0        UNKNOWN                                      
          102/102                   (Unspecified)                            D  No         Yes            0        UNKNOWN       


STEP -2 Installation of kamailio.

 2a.Configure it as registrar and register user.
     2a(1))Create two users 1000, 1001 in Kamilio and make calls b/w them
 
 
             apt-get install gnupg2 mariadb-server curl unzip -y
             apt-get install kamailio kamailio-sqlite-modules sqlite3 kamailio-mysql-modules
             vim /etc/kamailio/kamctlrc
             ## your SIP domain
             SIP_DOMAIN=your-server-ip
             DBENGINE=MYSQL
             ## database host
             DBHOST=localhost


             save teh file and restart kamailio

              kamdbctl create
              It will start creating teh tables and config enter the mysql password to proceed further.

              vim /etc/kamailio/kamailio.cfg
              Add the following lines below #!KAMAILIO.

            #!define WITH_MYSQL
            #!define WITH_AUTH
            #!define WITH_USRLOCDB
            #!define WITH_ACCDB

            save the file adn restart kamailio

            systemctl restart kamailio
            systemctl status kamailio

            sudo vim /etc/default/kamailio

            Set the following variables to have the Kamailio service run as under the Kamailio user and group and to enable Kamailio to run

           RUN_KAMAILIO=yes

            USER=kamailio

            GROUP=kamailio

            save now the file

            Adding a User to Kamailio
            Next, add a user to Kamailio - this is the account the softphone will use to register with Kamailio

            $ sudo kamctl add 1000 1000pass
            $ sudo kamctl add 1001 1001pass

            Now users are ready to register with kamailio same can be register and check with any of the softphone clinets and also you can make call                   between the phones now.
            
            
            2b)Use dispatcher module to load balance inbound calls b/w two asterisk
            
            Enable the module 
           
            loadmodule "dispatcher.so"
            
            # ----------------- setting module-specific parameters ---------------


          modparam("dispatcher", "ds_ping_interval", 60)
          modparam("dispatcher", "ds_ping_latency_stats", 60)
          modparam("dispatcher", "ds_latency_estimator_alpha", 900)
          modparam("dispatcher", "db_url", DBURL)
          modparam("dispatcher", "table_name", "dispatcher")
          modparam("dispatcher", "flags", 2)


          ####### Routing Logic ########

          request_route {

          if(method=="INVITE"){
             ds_select_dst(1, 4);    #Get a random up destination from dispatcher
             route(RELAY);           #Route it
          }



          from the shell we’ll use kamctl to add a new dispatcher entry.

          kamctl dispatcher add 1 sip:Serverip1:5060 0 0 '' 'Asterisk Gateway 1'
          kamctl dispatcher add 1 sip:Serverip2:5060 0 0 '' 'Asterisk Gateway 2' 


          kamctl dispatcher reload


          Now the call which are initaing will go to each asterisk server random basics .


2c. Filter inbound calls only from a certain IP source add belwo configs



              #!KAMAILIO
              #!define WITH_MYSQL
              #!define WITH_AUTH
              #!define WITH_USRLOCDB
              #!define WITH_ACCDB
              #!define WITH_IPAUTH
              #!define WITH_DEBUG
              #!define WITH_PERMISSIONS


              #!ifdef WITH_AUTH
              loadmodule "auth.so"
              loadmodule "auth_db.so"
              #!ifdef WITH_IPAUTH
              loadmodule "permissions.so"
              #!endif
              #!endif


              #!ifdef WITH_IPAUTH
              modparam("permissions", "db_url", DBURL)
              modparam("permissions", "db_mode", 1)
              #!endif


              request_route {

              route(REQINIT);

              if (allow_source_address("200")) {
                      xlog("Coming from address group 200");
              }else if (allow_source_address("250")) {
                      xlog("Coming from address group 250");
              }else{
                      sl_reply("401", "Address not authorised");
                      exit;
              }


              Now save the file and restart kamailio
              allow the tarffic from single IP

              kamctl address add 250 10.8.203.139 32 5060 AllowOneIP
              kamctl address reload



2c.Filter inbound calls only from a certain IP range source.

          kamctl address add 200 192.168.1.0 24 5060 OfficeSubnet
          kamctl address reload



---------------------------------











