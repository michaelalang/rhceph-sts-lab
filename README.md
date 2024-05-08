# Red Hat Ceph Storage STS configuration

AWS STS (Security Token Service) [API](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) provides a comfortable way to avoid storing credentials in configuration files or environment variables like used with aws cli tool.
This Lab will guide you through the setup process step-by-step and will tackle following technologies:
* [LDAP](#create-a-ldap-instance), used as backend for user storage and credential based authentication
* [Red Hat Single Sign On](#create-a-sso-instance), used as web based integration flow for token layer authentication
* [Red Hat Ceph Storage](#deploy-red-hat-ceph-storage-system), used as S3 Storage frontend utilizing STS roles and policies for granting access

**NOTE**
```
This Lab is not supported by Red Hat in anyway 
and should also not be used as production implementation.
```

# Requirements

The Lab is based upon following requirements
* Red Hat Enterprise Linux 9 base [images](https://access.redhat.com/downloads/content/rhel)
* Red Hat Enterprise Linux 9 Workstation (hosting podman containers)
* access to registry.redhat.io (get your credentials at [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret))
* subscription-manager access to the repositories
	* rhceph-6-tools-for-rhel-9-x86_64-rpms
	* rhceph-7-tools-for-rhel-9-x86_64-rpms
*  aws cli (install [instructions](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
* KVM virtualization capabilities (for Red Hat Ceph Storage)
* DNS (dnsmasq) configuration pointing to the names of the instances deployed
* 150GB diskspace (100GB for Red Hat Ceph OSD storage, images and temporary data)
* +4 Cores
* +16GB Memory 

# Limitations
* the Lab is not meant for Production purpose
* Red Hat Ceph Storage will be deployed as Single Node instance 
* Red Hat Build of Keycloak will be deployed without a Database
* ds389 will be deployed without persistent storage
* even though all Containers can run rootless you need to set 

    sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80

**NOTE** rebooting any instance makes it neccessary to re-iterate through the setup steps.

# Lab Information
* all passwords are set to `changeme` 

## Create a self-signed CA
to ensure we have a `trusted` path between all involved components we are going to deploy our own CA and a certificate with following names to match:
* sso.example.com
* ldap.example.com
* rgw.example.com
* s3.example.com
* Execute the following steps to create the neccessary certificates

    ```
    openssl genrsa -out ca.key 4096
    openssl req -x509 -new -nodes -key ca.key -sha256 -days 30 -out ca.crt -subj '/CN=STS Root CA/C=US/ST=STS/L=STS/O=STS'
    openssl req -new -nodes -out tls.csr -newkey rsa:4096 -keyout tls.key -subj '/CN=*.example.com/C=US/ST=STS/L=STS/O=STS'
    cat <<'EOF'> tls.v3.ext
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = sso.example.com
    DNS.2 = ldap.example.com
    DNS.3 = rgw.example.com
    DNS.4 = s3.example.com
    EOF
    openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 30 -sha256 -extfile tls.v3.ext
    ```

* Update you local trust bundle if you want to use your Workstation for connecting to the services

    ```
    sudo cp ca.crt /etc/pki/ca-trust/source/anchors/
    sudo update-ca-trust 
    ```

### Create a Java truststore
To ensure SSO to LDAP connections will be trusted, we need to create our custom Java truststore.
* Initialize a new truststore

    ```
    keytool -keystore examplestore -genkey -alias example
    ```

* Import the example CA

    ```
    keytool -import -keystore examplestore -file ca.crt -alias exampleCA
    ```

* Import the wildcard Certificate

    ```
    keytool -import -keystore examplestore -file tls.crt -alias wildcard-example
    ```

This truststore will later be used when initializing the SSO deployment.

## Create a Virtual System for Red Hat Ceph Storage

Even though Red Hat Ceph Storage 7 is shipped containerized, we utilize a Virtual Machine to have a clean separation between the Hypervisor resources and the Lab required resources.
Download the image [Red Hat Enterprise Linux 9.4 KVM](https://access.cdn.redhat.com/content/origin/files/sha256/d3/d362e72518a2d7415d850b8177c814f6fd87f42ca1640bda17e98855e1d6baad/rhel-9.4-x86_64-kvm.qcow2?user=d03bc7cd3b0bc44085fc19e06bfad453&_auth_=1715093182_0c67e667b5d0862e6633cae008814081) Guest image from Red Hat.

* Prepare your Workstation to be a Hypervisor for Containers and Virtual machines.

    ```
    sudo dnf -y install \
      virt-manager \
      virt-install \
      podman \
      skopeo \
      guestfs-tools \
      xorriso
    ```

* Create virt-customize instructions to prepare Red Hat Ceph installation

    ```
    # choose your Ceph version by exporting the approriate repository
    export CEPH_VERSION=rhceph-6-tools-for-rhel-9-x86_64-rpms
    export CEPH_VERSION=rhceph-7-tools-for-rhel-9-x86_64-rpms
    
    # export your subscription-manager credentials 
    export USERNAME=...
    export PASSWORD=...
    	    
    cat <<EOF> cmd.txt
    subscription-manager register \
      --username=${USERNAME} \
      --password=${PASSWORD} \
      --name=ceph \
      --auto-attach \
      --servicelevel='Self-Support'
    subscription-manager repos --enable ${CEPH_VERSION}
    dnf install cephadm podman -y 
    useradd -G wheel cephadmin
    sed -i -e " s#^%wheel.*#%wheel     ALL=(ALL)       NOPASSWD: ALL#; " /etc/sudoers
    mkdir /root/.config/containers -p 
    runuser	-u cephadmin -- ssh-keygen -N '' -f /home/cephadmin/.ssh/id_rsa
    EOF
    ```

    The cmd.txt file will be executed during the customization process and ensure we have the neccessary packages and configurations in place to just bootstrap a Singel Node Ceph Cluster in partticular it does
	
	* Registeres the System with subscription-manager
	* Enables the Ceph repository (v6 or v7)
	* Installs packages: 
		* cephadm
		* podman 
	* Adds an unprivileged user for Ceph called `cephadmin` into the Group `wheel`
	* Updates sudoers to not prompt for passwords for members of Group `wheel`
	* Creates container related folders (auth.json) 
	* Creates a RSA ssh key for Ceph deployments (necessary even in Single Node) 
* Store your registry access https://console.redhat.com/openshift/install/pull-secret token as `auth.json` in the same directory as your Red Hat Enterprise Linux 9.4 KVM Guest image.
* Customize the image with the following command
	
    ```
    virt-customize -a rhel-9.4-x86_64-kvm.qcow2 \
      --run cmd.txt \
      --copy-in auth.json:/root/.config/containers/
    ```

    * You can as well include bootstrap and rollout definitions in the base image by extending  the command with multiple `--copy-in` statements
* Sysprep the image to be re-useable 

    ```
    virt-sysprep -a rhel-9.4-x86_64-kvm.qcow2 \
        --operations machine-id,bash-history,logfiles,tmp-files,net-hostname,net-hwaddr
    ```
    
* Create a cloud-init ISO for customizing login and access accordingly

    ```	
    touch meta-data
    cat <<EOF> user-data
    #cloud-config

    user: cloud-user
    password: changeme

    chpasswd:
      expire: False
       
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_rsa.pub)
    EOF
    
    genisoimage -input-charset utf-8 -output cloudinit.iso -volid cidata -joliet -rock user-data meta-data
    ```

	* The default user will be named `cloud-user`
	* The default password will be `changeme`
	* If existing your SSH public key will be added for access to the cloud-user 

* Create a Virtual Machine with the customized image and two additional empty disk and the cloudinit ISO
	* You might need to adjust the qemu bridge configuration if you hit a `access denied by acl file` error

    ```
    # export the name of the Bridge you want to use
    export bridge=br0
    echo "allow ${bridge}" >>/etc/qemu-kvm/bridge.conf        
       
    # initialize the Virtual Machine
    virt-install \
      --vcpus 4 \
      --memory 16384 \
      --name ceph \
      --import \
      --disk rhel-9.4-x86_64-kvm.qcow2 \
      --disk size=50 \
      --disk size=50 \
      --network network=${bridge},model=virtio \
      --disk cloudinit.iso,device=cdrom \
      --os-variant rhel9.3
    ```

* Access the Virtual Machine as `cloud-user` with password `changeme`
	* You can access the System through SSH or console 
	* You can grant `root` access by 
		* updating the /etc/ssh/sshd_config 
		* updating the /root/.ssh/authorized_keys accordingly
		

            ```
            sudo sed -i -e ' s/#PermitRootLogin.*/PermitRootLogin yes/; ' /etc/ssh/sshd_config
            sudo systemctl restart sshd 
            sudo cp ~cloud-user/.ssh/authorized_keys /root/.ssh/authorized_keys
            ```

### Deploy Red Hat Ceph Storage System
* Access the Virtual System and escalate privileges
* Set the hostname of the system

    ```
    sudo hostnamectl hostname ceph.example.com
    ```

	*	**NOTE** you need to re-login to reflect the change on the shell
* Copy the Certificates to the Red Hat Ceph Storage

    ```
    scp ca.crt tls.crt tls.key cloud-user@ceph.example.com:
    ```

* update the trust bundle on the Ceph Host

    ```	 
    sudo cp ca.crt /etc/pki/ca-trust/source/anchors/ 
    sudo update-ca-trust
    ```

* Escalate the `cloud-user` privileges

    ```
    sudo -i
    ```


* Export the Host IP for bootstrapping

    ```
    export CEPHIP=$(ip -br a | awk '/eth0/ { print $3 }' | cut -f1 -d'/')
    ```

* Export the Hostname for bootstraping

    ```
    export RGWHOST=$(hostname)
    ```

* Export the Certifiacate and Key for RGW
   
    ```
    export CERTIFICATE=$(cat ~cloud-user/tls.crt ~cloud-user/tls.key | sed -e ' s#^#    #g;')
    ```

   * Create your rollout.yaml file 
   

		 # ensure to update the device section according to your virtualization bus
		 cat <<EOF> rollout-tmpl.yaml
		 service_type: host
		 addr: ${CEPHIP}
		 hostname: ${RGWHOST}
		 labels:
		 - mon
		 - mgr
		 - osd
		 - rgw
		 ---
		 service_type: mon
		 placement:
		   label: mon
		 ---
		 service_type: mgr
		 placement:
		   label: mgr
		 ---
		 service_type: rgw
		 service_id: default
		 placement:
		   label: rgw
		 spec:
		   rgw_frontend_port: 443
		   rgw_frontend_ssl_certificate:  |
         ${CERTIFICATE} 
           ssl:  true
           extra_container_args:
             - -v
             - /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem:/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem
		 ---
		 service_type: osd
		 service_id: default_osd_group
		 placement:
		   hosts:
		     - ${RGWHOST}
		 data_devices:
		   paths:
		     - /dev/sdb   # update to your deployment disk system
		     - /dev/sdc   # update to your deployment disk system
		 EOF
* Create your Ceph cluster through cephadm

	  # expand variables in the yaml template to a Ceph orchestrator yaml
	  cat rollout-tmpl.yaml | envsubst > rollout.yaml

	  # bootstrap the Ceph cluster
	  cephadm bootstrap \
		--mon-ip ${CEPHIP} \
		--initial-dashboard-user admin \
		--initial-dashboard-password changeme \
		--ssh-private-key /home/cephadmin/.ssh/id_rsa \
		--ssh-public-key /home/cephadmin/.ssh/id_rsa.pub \
		--ssh-user cephadmin \
		--dashboard-password-noupdate \
		--allow-fqdn-hostname \
		--allow-overwrite \
		--apply-spec ./rollout.yaml \
		--single-host-defaults
* Execute a cephadm shell to configure following STS
	* OIDC user for the OIDC provider
	* STS Key
	* RGW to use STS
	* STS Roles
	* Role Policies
	
		  cephadm shell 
		  # export OIDC access and secret 
		  export OIDCUID=oidc
		  export OIDCPWD=oidc
		
		  # create the RGW user for the OpenID Provider configuration
		  radosgw-admin user create \
		    --uid=${OIDCUID} \
		    --display-name="OpenID Provider user" \
		    --access-key=${OIDCUID} \
		    --secret-key=${OIDCPWD}

		  # Grant the oidc-provider CAPS (* for Lab simplification)
		  radosgw-admin caps add \
		    --uid=${OIDCUID} \
		    --caps="oidc-provider=*"

          # Create a 16 char random string as STS Key 
          ceph config set client.rgw rgw_sts_key $(tr -dc A-Za-z0-9 </dev/urandom | head -c 16; echo)

		  # Configure RGW to use STS as authentication mechanism
		  ceph config set client.rgw rgw_s3_auth_use_sts true

          # Restart the RGW instance as it doesn't pick up the changes properly otherwise
          ceph orch restart rgw.default

	* Creating a Role for Administrator access through STS and `group` assignment
	Depending on our SSO implementation we need to understand the `Condition` matching. With LDAP as backend and SSO federating those, it is mandatory to understand that settings in the SSO realm affect the `Condition` matching.
		* When federating LDAP groups in SSO, we can select the Groups to be mapped to the Tokens with `Full group path` or without. The `Full group path` will expose the groups in the token with a leading slash `/`. Nested groups are the reason for that as they will for example show up like `/admin/rgwadmin/engineering` which reflects three groups `admin` sub group `rgwadmin` and a subgroup of rgwadmin `engineering`.
		* For the Lab we do not utilize nested groups and we syntax every configuration to **NO** `Full group path` configurations.
		* execute follwing in our cephadm shell to create the role and policy

			  export SSO=sso.example.com
			  export SSOREALM=ceph
				  
			  cat <<EOF | python3 -m json.tool --compact > /tmp/role.json
			  {
					 "Version": "2012-10-17",
					 "Statement": [
					   {
					     "Effect": "Allow",
					     "Principal": {
					       "Federated": [
					         "arn:aws:iam:::oidc-provider/${SSO}/realms/${SSOREALM}"
					       ]
					     },
					     "Action": [
					       "sts:AssumeRoleWithWebIdentity"
					     ],
					     "Condition": {
					       "StringLike": { "${SSO}/realms/${SSOREALM}:groups":["rgwadmins"] }
					     }
					   }
					 ]
			  }
			  EOF
		* the command will create a compact one-line json content as radosgw-admin does not accept a json-nice formatted input
	* execute follwing in our cephadm shell to create the policy for our admin role
	

		  cat <<EOF | python3 -m json.tool --compact > /tmp/policy.json
		  {
			"Version": "2012-10-17",
			"Statement": [
			  {
			   "Effect": "Allow",
			   "Action": [ "s3:*" ],
			   "Resource": "arn:aws:s3:::*"
			  }
			]
		  }
		  EOF
		  

		* that policy will grant all `rgwadmins` full access
* we now add the role and policy to our RGW instance

	  radosgw-admin role create \
	    --role-name rgwadmins  \
	    --assume-role-policy-doc=$(cat /tmp/role.json)
	  radosgw-admin role policy put \
	    --role-name=rgwadmins \
	    --policy-name=admin \
	    --policy-doc=$(cat /tmp/policy.json)
			
* verify that everything is setup correctly by executing

	  radosgw-admin role list






## Create a LDAP instance

* Start a LDAP instance as follows

	  podman run -d \
		  --name ds389 \
		  -p 389:3389 \
		  -p 636:3636 \
		  --rm \
		  -v $(pwd)/tls.crt:/data/tls/server.crt:z \
		  -v $(pwd)/tls.key:/data/tls/server.key:z \
		  -v $(pwd)/ca.crt:/data/tls/ca/ca.crt:z \
		  -e DS_DM_PASSWORD=changeme \
		  -e DS_SUFFIX_NAME=dc=example,dc=com \
		  docker.io/389ds/dirsrv:latest
	* **HINT** the initial startup will take a minute
* Create a Directory content LDIF file

	  cat <<'EOF'> example.com.ldif
	  dn: dc=example,dc=com
	  objectClass: top
	  objectClass: domain
	  dc: example
	  description: dc=example,dc=com

	  dn: ou=People,dc=example,dc=com
	  ou: People
	  objectClass: top
	  objectClass: organizationalUnit

	  dn: ou=Groups,dc=example,dc=com
	  ou: Groups
	  objectClass: top
	  objectClass: organizationalUnit

	  dn: cn=Lisa Collins,ou=People,dc=example,dc=com
	  cn: Lisa Collins
	  uid: lisa
	  uidNumber: 90698
	  gidNumber: 90698
	  loginShell: /sbin/nologin
	  homeDirectory: /home/huynhbrittany
	  displayName: Lisa Collins
	  shadowExpire: 1
	  shadowInactive: 1
	  shadowLastChange: 19176
	  shadowMax: 99999
	  shadowMin: 0
	  shadowWarning: 7
	  objectClass: top
	  objectClass: nsPerson
	  objectClass: nsAccount
	  objectClass: nsOrgPerson
	  objectClass: posixAccount
	  objectClass: shadowaccount
	  objectClass: inetuser
	  userPassword:: e1BCS0RGMl9TSEEyNTZ9QUFBSUFBSThXb0N4UFhUOUdzNUxMM1QvZEM5eE5LZS9
	   KZmNBT3E1OEpUQU56dkRJTG9XYkJCR1Roc0NrdkpLOW02TDM4M1JkVXVqdGw5WUtjNWgrUDVKTHNS
	   NU5tNE1mQlYrT3VHT1diUHNQOUZsdFdleHU3UjIzTldSRi9hRys5ZDM4azJRaWRVOGhXaitYYmhqe
	   GppTlROS1drVXcvT2lzTzVBUWdxRTB6eU1KWXJ3a1RqdGk5QllFclREb2NzYnl4OHgyc1RRcGlkV2
	   tBKy9Uc1BWemc1dXZ0UElqZGNDWEFhSE54T28wTnUvcmNuZlkrSm12MUN1R3BNdVpYWDY5RUp4Nis
	   wL3M2QkZDNnVNZC9vWWVQVUt6b2pmakRnT3ZKUnk0N3BVVzFCUVM4Q3o0VzhSWDZPaFlCVFRTalFV
	   dHlqaklUb3Y4KzlJVkRwSitJNlk2Y3RRZ0NLWDlHR2VIWGdPMzJJTmE3R1lmb2w3TStuUE83UDVTd
	   2s3bmVpQmJWUFNud3FwSG45eU5KK0syNWxKQVRIajZWTE9CYnhzVWxpQ3dKeE8wd1h3cE9U

	  dn: cn=Kimberly Weeks,ou=People,dc=example,dc=com
	  cn: Kimberly Weeks
	  uid: kimberly
	  uidNumber: 93578
	  gidNumber: 93578
	  loginShell: /sbin/nologin
	  homeDirectory: /home/melissa80
	  displayName: Kimberly Weeks
	  shadowExpire: 1
	  shadowInactive: 1
	  shadowLastChange: 19176
	  shadowMax: 99999
	  shadowMin: 0
	  shadowWarning: 7
	  objectClass: top
	  objectClass: nsPerson
	  objectClass: nsAccount
	  objectClass: nsOrgPerson
	  objectClass: posixAccount
	  objectClass: shadowaccount
	  objectClass: inetuser
	  userPassword:: e1BCS0RGMl9TSEEyNTZ9QUFBSUFBSThXb0N4UFhUOUdzNUxMM1QvZEM5eE5LZS9
	   KZmNBT3E1OEpUQU56dkRJTG9XYkJCR1Roc0NrdkpLOW02TDM4M1JkVXVqdGw5WUtjNWgrUDVKTHNS
	   NU5tNE1mQlYrT3VHT1diUHNQOUZsdFdleHU3UjIzTldSRi9hRys5ZDM4azJRaWRVOGhXaitYYmhqe
	   GppTlROS1drVXcvT2lzTzVBUWdxRTB6eU1KWXJ3a1RqdGk5QllFclREb2NzYnl4OHgyc1RRcGlkV2
	   tBKy9Uc1BWemc1dXZ0UElqZGNDWEFhSE54T28wTnUvcmNuZlkrSm12MUN1R3BNdVpYWDY5RUp4Nis
	   wL3M2QkZDNnVNZC9vWWVQVUt6b2pmakRnT3ZKUnk0N3BVVzFCUVM4Q3o0VzhSWDZPaFlCVFRTalFV
	   dHlqaklUb3Y4KzlJVkRwSitJNlk2Y3RRZ0NLWDlHR2VIWGdPMzJJTmE3R1lmb2w3TStuUE83UDVTd
	   2s3bmVpQmJWUFNud3FwSG45eU5KK0syNWxKQVRIajZWTE9CYnhzVWxpQ3dKeE8wd1h3cE9U

	  dn: cn=Kenneth Clay,ou=People,dc=example,dc=com
	  cn: Kenneth Clay
	  uid: kenneth
	  uidNumber: 35153
	  gidNumber: 35153
	  loginShell: /sbin/nologin
	  homeDirectory: /home/austinmiller
	  displayName: Kenneth Clay
	  shadowExpire: 1
	  shadowInactive: 1
	  shadowLastChange: 19176
	  shadowMax: 99999
	  shadowMin: 0
	  shadowWarning: 7
	  objectClass: top
	  objectClass: nsPerson
	  objectClass: nsAccount
	  objectClass: nsOrgPerson
	  objectClass: posixAccount
	  objectClass: shadowaccount
	  objectClass: inetuser
	  userPassword:: e1BCS0RGMl9TSEEyNTZ9QUFBSUFBSThXb0N4UFhUOUdzNUxMM1QvZEM5eE5LZS9
	   KZmNBT3E1OEpUQU56dkRJTG9XYkJCR1Roc0NrdkpLOW02TDM4M1JkVXVqdGw5WUtjNWgrUDVKTHNS
	   NU5tNE1mQlYrT3VHT1diUHNQOUZsdFdleHU3UjIzTldSRi9hRys5ZDM4azJRaWRVOGhXaitYYmhqe
	   GppTlROS1drVXcvT2lzTzVBUWdxRTB6eU1KWXJ3a1RqdGk5QllFclREb2NzYnl4OHgyc1RRcGlkV2
	   tBKy9Uc1BWemc1dXZ0UElqZGNDWEFhSE54T28wTnUvcmNuZlkrSm12MUN1R3BNdVpYWDY5RUp4Nis
	   wL3M2QkZDNnVNZC9vWWVQVUt6b2pmakRnT3ZKUnk0N3BVVzFCUVM4Q3o0VzhSWDZPaFlCVFRTalFV
	   dHlqaklUb3Y4KzlJVkRwSitJNlk2Y3RRZ0NLWDlHR2VIWGdPMzJJTmE3R1lmb2w3TStuUE83UDVTd
	   2s3bmVpQmJWUFNud3FwSG45eU5KK0syNWxKQVRIajZWTE9CYnhzVWxpQ3dKeE8wd1h3cE9U
	  
	  dn: cn=rgwadmins,ou=Groups,dc=example,dc=com
	  objectClass: top
	  objectClass: groupOfNames
	  cn: rgwadmins
	  member: cn=Lisa Collins,ou=People,dc=example,dc=com

	  dn: cn=rgwwriters,ou=Groups,dc=example,dc=com
	  objectClass: top
	  objectClass: groupOfNames
	  cn: rgwwriters
	  member: cn=Kimberly Weeks,ou=People,dc=example,dc=com

	  dn: cn=rgwreaders,ou=Groups,dc=example,dc=com
	  objectClass: top
	  objectClass: groupOfNames
	  cn: rgwreaders
	  member: cn=Kenneth Clay,ou=People,dc=example,dc=com
	  EOF
* Execute following command to create the Backend for the LDAP data

	  podman exec -ti ds389 dsconf localhost \
	    backend create \
	    --suffix dc=example,dc=com \
	    --be-name userRoot

* Add the LDIF to the LDAP backend

   

      LDAPTLS_CACERT=ca.crt ldapadd \
         -x -H ldaps://ldap.example.com:636 \
         -D 'cn=Directory Manager' \
         -w changeme \
         -f example.com.ldif

## Create a SSO instance

* Create your SSO instance

    **NOTE** you cannot run this in your homeDirectory due to SELinux prevention. Create a subfolder with the necessary files 

    ```
    # ensure Keycloak can read the cert key file (rootless issue)
    chmod +r tls.key 
    podman run --rm --userns keep-id --name sso -d -p 443:8443 -p 80:8080 \
	    -e KC_HOSTNAME=sso.example.com \
	    -e KD_HOSTNAME_ADMIN_URL=https://sso.example.com:443 \
	    -e KC_HTTP_ENABLED=true \
	    -e KC_HTTP_PORT=8080 \
	    -e KC_HTTPS_PORT=8443 \
	    -e KC_HTTPS_CERTIFICATE_FILE=/mnt/certificates/tls.crt \
	    -e KC_HTTPS_CERTIFICATE_KEY_FILE=/mnt/certificates/tls.key \
	    -e KC_PROXY=passthrough \
	    -e KEYCLOAK_ADMIN=admin \
	    -e KEYCLOAK_ADMIN_PASSWORD=changeme \
	    -v $(pwd):/mnt/certificates/:z \
	    registry.redhat.io/rhbk/keycloak-rhel9:24-7 start \
	       --optimized \
	       --spi-truststore-file-file=/mnt/certificates/examplestore \
	       --spi-truststore-file-password=changeme \
	       --spi-truststore-file-hostname-verification-policy=ANY
    ```

### Configure a Realm and Client for STS
* Login to SSO at https://sso.example.com with credentials `admin` and `changeme`
* Select the `Create realm` button
	* Enter the name `ceph` as `Realm name`
* Select `Realm settings`
	* Select the tab `Keys`
		* Select the `Add providers` section under the list of tabs
		* Select `rsa-geenerate` (RSA signature keys ...) Provider
		* Update the `Priority` to `105` to be the first listed when returning tokens
* Select `User Federation` in the Navigation Bar on the left
	* Select `Add Ldap providers`
		* Select `Red Hat Directory Server` in `Vendor` selection
		* Enter `ldaps://ldap.example.com:636` in the `Connection URL` field
			* Click `Test connection` to ensure SSL trustchain is working
		* Enter `cn=Directory Manager` in the `Bind DN` field
		* Enter `changeme` in the `Bind credentials` field
			* Click `Test authentication` to ensure authentication is working 
		* Select `READ_ONLY` in the `Edit mode` field
		* Enter `ou=People,dc=example,dc=com` in the `Users DN` field
		* Enter `nsPerson` in the `User object classes` field as only value
		* Select `Subtree` in the `Search scope` field
	* Click `Save` to configure the LDAP federation accordingly
	* Click `ldap` in the List of Federation providers
		* in the `Action` dropbox at the top corner right select `Sync all users`
		* Select the `Mappers` Tab
			* Click `Add mapper` to create a new group mapper
				* Enter `groups` ini the `Name` field
				* Select `group-ldap-mapper` in the `Mapper type` field
				* Enter `ou=Groups,dc=example,dc=com` in the `LDAP Roles DN` field
				* Click `Save` to store the configuration
				* Select `groups` from the list of Mappers
					* in the `Action` dropbox at the top corner right select `Sync LDAP groups to Keycloak`
	* Select `Clients` in the Navigation Bar on the left
		* Click `Create client` 
			* Enter `ceph` in the `Client ID` field
				* Click `Next`
			* Turn on `Client authentication`
			* Select `Service accounts roles` in the Authentication flow section
				* Click `Next`
			* Enter `*` in the `Valid redirect URIs` field
			* Enter `*` in the `Valid post logout redirect URIs` field
			* Enter `*` in the `Web origins` field
				* Click `Next`
		* Select the `Credentials` tab to retrieve the `Client Secret` value 
			* Store this secret for the scripts used later
		* Select the `Client scopes` tab 
			* Select the `ceph-dedicated` Scope (Dedicated scope and mappers for this client)
				* Click `Add mapper` 
					* Select `By configuration`
						* Select `User Property`
							* Enter `username` in the `Name` field
							* Enter `username` in the `Property` field
							* Enter `sub` in the `Token Claim Name`
								* Click `Save`
			* Select the `ceph-dedicated` Scope (Dedicated scope and mappers for this client)
				* Click `Add mapper` 
					* Select `By configuration`
						* Select `Group Membership` 
                                                * Enter `groups` in the `Name` field
						* Enter `groups` in the `Token Claim Name`
						* **Turn off** `Full group path` 
							* Click `Save`
			* Select the `ceph-dedicated` Scope (Dedicated scope and mappers for this client)
				* Click `Add mapper` 
					* Select `By configuration`
						* Select `Audience`
							* Enter `audience` in the `Name` field
							* Select `ceph` in the `Included Client Audience` dropdown
							* Turn on `Add to ID token`
								* Click `Save`

## Create an OIDC-id connect provider in RGW
* Export the neccessary configuration

	  export SSO=sso.example.com
	  export SSOREALM=ceph

* Retrieve the Signing fingerprint

	  export FPRINT=$(curl --cacert ca.crt -s $(curl --cacert ca.crt -s https://${SSO}/realms/${SSOREALM}/.well-known/openid-configuration | jq -r .jwks_uri ) | \
	                jq -r ' .keys[] | select(.use=="sig") | .x5c[] |
	                      ["-----BEGIN CERTIFICATE-----", 
	                       .,
	                       "-----END CERTIFICATE-----"] |
	                      join("\n")' | \
	                openssl x509 -noout -fingerprint | \
	                cut -f2 -d'=' | \
	                sed -e 's#:##g; ')

	* **NOTE** it is mandatory to understand the the returned content will contain more than one certificate data. One is used for encrypting and the other one is used for signatures. In the SSO configuration I guided you through setting the Priority of the Signature Certificate higher than the encryption Certificate to get for STS and for Clients always the expected Certificate first. The Snipped above handles it through `select(.use=="sig")` and you will get RGW error log entries with `invalid padding` and `An error occurred (Unknown) when calling the AssumeRoleWithWebIdentity operation: Unknown` Client errors if the Certificate is the wrong one. 
	* **NOTE** if your TLS chain is broken you will see `-13` errors in RGW's log.
	* Turn on RGW debugging with

		  cephadm shell ceph config set client.rgw debug_rgw 20/5
		  cephadm shell ceph config set client.rgw debug_ms 1/5
* Create the Open-id-connect Provider in RGW

    ```
    export AWS_CA_BUNDLE=ca.crt 
    export AWS_REGION=${REGION} 
    export AWS_ACCESS_KEY_ID=oidc
    export AWS_SECRET_ACCESS_KEY=oidc
    export SSO=sso.example.com
    export SSOCLIENT=ceph
    export RGW=rgw.example.com
	  
    aws --endpoint-url https://${RGW} iam create-open-id-connect-provider \
	    --url https://${SSO}/realms/${SSOREALM} \
	    --thumbprint-list ${FPRINT} \
	    --client-id-list ${SSOCLIENT}
    ```

* Verify the Open-id-connect Provider

	  aws --endpoint-url https://${RGW} \
	    iam get-open-id-connect-provider \
	    --open-id-connect-provider-arn \
	      arn:aws:iam:::oidc-provider/${SSO}/realms/${SSOREALM}

	* **NOTE** when ever you have already utilized STS you need to unset `AWS_SESSION_TOKEN` to use `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` again.
		  
## Client setup for STS

### retrieve your OIDC access token 
* create a wrapper to simplify OAuth token retreival 

	  cat <<'EOF'> token
	  export SSO=sso.example.com
	  export SSOREALM=ceph
	  export CLIENT_SECRET=${CLIENT_SECRET:-"invalid"}
	  
	  USERNAME=$1
	  PASSWORD=$2
	  
	  KC_ACCESS_TOKEN=$(curl -s --cacert ca.crt -k -q -L -X POST "https://${SSO}/realms/${SSOREALM}/protocol/openid-connect/token" \
	  		   -H 'Content-Type: application/x-www-form-urlencoded' \
	  		   --data-urlencode "client_id=${SSOREALM}" \
	  		   --data-urlencode 'grant_type=password' \
	  		   --data-urlencode "client_secret=${CLIENT_SECRET}" \
	  		   --data-urlencode 'scope=openid' \
	  		   --data-urlencode "username=${USERNAME}" \
	  		   --data-urlencode "password=${PASSWORD}" | \
	  		jq -r .access_token)
	  
	  export KC_ACCESS_TOKEN
	  EOF

### retrieve your STS session and AWS access token 

* create a wrapper to simplify STS authentication and AWS credentials exchange

    ```
    cat <<'EOF'> sts-role
    #!/bin/bash
	  
    export AWS_CA_BUNDLE=ca.crt
    unset AWS_ACCESS_KEY_ID
    unset AWS_SECRET_ACCESS_KEY
    unset AWS_SESSION_TOKEN
    export AWS_EC2_METADATA_DISABLED=true
	  
    USERNAME=$1
    PASSWORD=$2
    ROLE=$3
    SESSION=${4:-"test"}
    RGW=rgw.example.com 
	  
    source token ${USERNAME} ${PASSWORD} ${ROLE}
	  
    IDM_ASSUME_ROLE_CREDS=$(aws sts assume-role-with-web-identity \
	  	--role-arn "arn:aws:iam:::role/${ROLE}" \
	  	--role-session-name ${SESSION} \
	  	--endpoint=https://${RGW} \
	  	--web-identity-token="${KC_ACCESS_TOKEN}")
    export AWS_ACCESS_KEY_ID=$(echo $IDM_ASSUME_ROLE_CREDS | jq -r .Credentials.AccessKeyId)
    export AWS_SECRET_ACCESS_KEY=$(echo $IDM_ASSUME_ROLE_CREDS | jq -r .Credentials.SecretAccessKey)
    export AWS_SESSION_TOKEN=$(echo $IDM_ASSUME_ROLE_CREDS | jq -r .Credentials.SessionToken)
    ```
    * type EOF manually please to avoid resolution during cat'ing

* **NOTE** the environment setting of `AWS_EC2_METADATA_DISABLED=true` disabled AWS default metadata resolution and speeds up the process by not asking for `http://169.254.169.254/latest/meta-data/...` 
* Source the `sts-role` with parameters `username` `password` STS `role` and optional a session identifier

    ```
    # ensure to export your Client_secret
    export CLIENT_SECRET=....

    # STS authentication with a specific session name "admin"
    source sts-role lisa changeme rgwadmins admin
	   
    # STS authentication with a default session name "test"
    source sts-role lisa changeme rgwadmins 
    ```

* Access your S3 API to create your bucket with the STS authentiation workflow

	  aws --endpoint https://${RGW} s3api create-bucket --bucket=testing
	* Upload some sample content into your bucket

		  aws --endpoint https://${RGW} s3 cp ca.crt s3://testing/

	* List all content from your bucket

		  aws --endpoint https://${RGW} s3 ls s3://testing/

Congratulations, you successfully enabled STS on your RGW 

## Conclusion

Even though, STS isn't hard to implement the depending infrastructure is making the setup complicated and "human" error prone due to various small settings not being aligned between all the layers for authentication. 

I do definetly recommend to use a separate Realm and Client for integrating STS in an existing OIDC/SSO infrastructure as some changes might otherwise conflict and or break existing working flows. 

One of those examples is `preferred_username` vs `username`. 
Another one is that Role mappings might invalidate existing authentication flows.

It is also quite annoying that Client side wise the error is always the same no matter what is broken in the workflow.

Lack of human-reabable debug on the RGW side makes it even more difficult to understand which checkbox is missing.


## Debugging STS problems
STS is not hard to be setup but tricky to debug due to the lack of none Engineering responses in the error logs.  We have a [Bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=2268379) opened to improve that.
* Always turn on rgw debug logs with

      cephadm shell ceph config set client.rgw debug_rgw 20/5
      cephadm shell ceph config set client.rgw debug_ms 1/5

	* disable debugging after you have finished
	

          ceph config rm client.rgw debug_rgw
    	  ceph config rm client.rgw debug_ms

* Watch the journal output for and reduce noise of none related content

      journalctl --no-pager -f | \
        grep radosgw  | \
        egrep -ve '(ping cookie|osd_op|ping magic|lua background)'

* Double check the Signature Certificate Fingerprint 

      curl --cacert ca.crt -s \
        $(curl --cacert ca.crt -s \
           https://${SSO}/realms/${SSOREALM}/.well-known/openid-configuration | \
             jq -r .jwks_uri ) | \
                jq -r ' .keys[] | select(.use=="sig") | .x5c[] |
                      ["-----BEGIN CERTIFICATE-----", 
                        .,
                        "-----END CERTIFICATE-----"] |
                      join("\n")' | \
                openssl x509 -noout -fingerprint | \
                cut -f2 -d'=' | sed -e 's#:##g; '
	Check the response with the configured `open-id-connect` Fingerprint
	

      AWS_REGION=${REGION} \
      AWS_ACCESS_KEY_ID=${OIDCUID} \
      AWS_SECRET_ACCESS_KEY=${OIDCPWD} \
      aws --endpoint-url https://${RGW} \
        iam get-open-id-connect-provider \
        --open-id-connect-provider-arn \
        arn:aws:iam:::oidc-provider/${SSO}/realms/${SSOREALM} \
        jq -r '.ThumbprintList'

* Double check that your System have a valid SSL Chain of trust

      podman exec -ti $(podman ps | \
         awk ' /rgw-default/ { print $1 } ') \
           curl https://sso.example.com -I
   You should not see any SSL issue here ... 

* Double check our token and introspect token from OIDC 
	
      cat <<'EOF'> jwt 
      decode_base64_url() {
	    local len=$((${#1} % 4))
  	    local result="$1"
	    if [ $len -eq 2 ]; then result="$1"'=='
	    elif [ $len -eq 3 ]; then result="$1"'='
	    fi
	    echo "$result" | tr '_-' '/+' | openssl enc -d -base64
	  }

	  decode_jwt(){
	    decode_base64_url $(echo -n $1 | cut -d "." -f 2) | jq .
	  }
      EOF
      
      source jwt
      source token lisa changeme
      decode_jwt ${KC_ACCESS_TOKEN}
	* Ceph and STS requires following attributes to be presented
		* aud, shall be ceph for the Lab and represenst the Realm/Client
		* sub, represents the username from LDAP `uid`
		* groups, represents the mapper for the roles, check on prefixing chars like `/`
		* scope, shall contain `openid` to ensure it will be considered in scoping
		* iss, shall be the SSO base URI including the realm **NOTE** RHSSO 7 will have a path including `/auth` where RHBK will not
		

			  decode_jwt ${KC_ACCESS_TOKEN}
			  {
			    "exp": 1715163080,
			    "iat": 1715162780,
			    "jti": "7be09445-d59d-4cc3-bb38-31ec6ddc9863",
			    "iss": "https://sso.example.com/realms/ceph",
			    "aud": "ceph",
			    "sub": "lisa",
			    "typ": "Bearer",
			    "azp": "ceph",
			    "session_state": "e92becf1-9390-4f63-b07d-fdb88dff4c03",
			    "acr": "1",
			    "allowed-origins": [
			  	"*"
			     ],
			    "realm_access": {
			       "roles": [
			         "rgwadmins"
			       ]
			    },
			    "scope": "openid profile email",
			    "sid": "e92becf1-9390-4f63-b07d-fdb88dff4c03",
			    "email_verified": false,
			    "name": "Lisa Collins",
			    "groups": [
			      "rgwadmins"
			    ],
			    "preferred_username": "lisa",
			    "given_name": "Lisa Collins"
			  }
      * Ensure the Token is valid 
      
			# access token attribute "exp": 1715163080
			date  --date='@1715163080'


