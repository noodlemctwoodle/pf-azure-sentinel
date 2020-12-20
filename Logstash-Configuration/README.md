# pfSense > LogStash > Azure Sentinel

## Table of Contents

  - [Configure Ubuntu server](#ubuntu-2004-server-onprem)
  - [Install MaxMind and configuration](#install-maxmind-database)
  - [Install and configure Logstash](#logstash-configuration)
  - [Forwarding pfSense logs](#forwarding-pfsense-logs-to-logstash)
  - [Install Logstash Azure Analytics plugin](#install-log-analytics-plugin)
  - [View logs in Azure Sentinel](#view-pfsense-logs-in-azure-sentinel)
  - [Query the logs in Azure Sentinel](#query-logs-in-azure-sentinel)

### Ubuntu 20.04 Server onPrem
  
1. Install Ubuntu Server 20.04 on a Virtual Machine or Computer and update the OS

        sudo apt update; sudo apt upgrade -y

2. Disabling Swap - Swapping should be disabled for performance and stability. (optional)

       sudo swapoff -a

3. Configuration Date/Time Zone

    - The box running this configuration will reports firewall logs based on its clock. The command below will set the timezone to Eastern Standard Time (EST).
    - To view available timezones type `sudo timedatectl list-timezones`

          sudo timedatectl set-timezone Europe/London

4. Download and install the public GPG signing key

        wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

5. Download and install apt-transport-https package

        sudo apt install apt-transport-https

6. Add LogstashRepositories (version 7+)

        echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list

7. Install Java 14 LTS

       sudo apt install openjdk-14-jre-headless

### Install MaxMind Database

1. Add MaxMind Repository (Ubuntu Only)

       sudo add-apt-repository ppa:maxmind/ppa

2. Install MaxMind (Ubuntu Only)

       sudo apt install geoipupdate

3. Configure MaxMind

    Create a [MaxMind Account](https://www.maxmind.com/en/geolite2/signup)

    Login to your MaxMind Account; navigate to "My License Key" under "Services" and Generate new license key

        sudo nano /etc/GeoIP.conf

4. Modify lines 7 & 8 as follows (without < >):

        AccountID <Input Your Account ID>
        LicenseKey <Input Your LicenseKey>

5. Modify line 13 as follows:

        EditionIDs GeoLite2-City GeoLite2-Country GeoLite2-ASN

6. Modify line 18 as follows:

        DatabaseDirectory /usr/share/GeoIP/

7. Download Maxmind Databases

        sudo geoipupdate

8. Add cron (automatically updates Maxmind everyweek on Sunday at 1700hrs)

        sudo nano /etc/cron.weekly/geoipupdate

9. Add the following and save/exit

        00 17 * * 0 geoipupdate

### Logstash Configuration

1. Install Logstash

        sudo apt update && sudo apt install logstash -y

2. Configuration
Create Required Directories

        sudo mkdir /etc/logstash/conf.d/{databases,patterns,templates}

3. Download the following configuration files (Required)

        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/01-inputs.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/02-types.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/03-filter.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/05-firewall.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/07-interfaces.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/10-apps.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/30-geoip.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/45-cleanup.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/50-outputs.conf -P /etc/logstash/conf.d/

4. Download the following configuration files (Optional)

        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/35-rules-desc.conf -P /etc/logstash/conf.d/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/36-ports-desc.conf -P /etc/logstash/conf.d/

5. Download the grok pattern (Required)

        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/patterns/pfelk.grok -P /etc/logstash/conf.d/patterns/

6. Download the Database(s) (Optional)

        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/databases/rule-names.csv -P /etc/logstash/conf.d/databases/
        sudo wget https://raw.githubusercontent.com/noodlemctwoodle/pfsense-azure-sentinel/main/Logstash-Configuration/etc/logstash/conf.d/databases/service-names-port-numbers.csv -P /etc/logstash/conf.d/databases/

7. Configure Firewall Rule Database (Optional)
    Go to your pfSense GUI and go to Firewall -> Rules.

    - Ensure the rules have a description, this is the text you will see in Azure Sentinel.
    - Block rules normaly have logging on, if you want to see good traffic also, enable logging for pass rules.

    Extract rule descriptions with associated tracking number (Optional & pfSense Only)

    In pfSense and go to diganotics -> Command Prompt

    Enter the following command in the execute shell command box and click the execute button

        pfctl -vv -sr | grep USER_RULE | sed 's/@\([^(]*\).*"USER_RULE: *\([^"]*\).*/"\1","\2"/' | sort -t ' ' -k 1,1 -u

    The results will look something like this:

        "55","NAT Redirect DNS"
        "56","NAT Redirect DNS"
        "57","NAT Redirect DNS TLS"
        "58","NAT Redirect DNS TLS"
        "60","BypassVPN"

    Copy the entire results to your clipboard and past within the rule-names.csv as follows:

        "Rule","Label"
        "55","NAT Redirect DNS"
        "56","NAT Redirect DNS"
        "57","NAT Redirect DNS TLS"
        "58","NAT Redirect DNS TLS"
        "60","BypassVPN"

8. Update the logstash configuration (Optional & pfSense Only)

    Go back to the server you installed Logstash.

        sudo nano /etc/logstash/conf.d/databases/rule-names.csv

9. Paste the the results from pfSense into the first blank line after "0","null"

    Example:

        "0","null"
        "1","Input Firewall Description Here

    You must repeat step 1 (Rules) if you add new rules in pfSense and then restart logstash

10. Update firewall interfaces

    Amend the 05-firewall.conf file

        sudo nano /etc/logstash/conf.d/05-firewall.conf

    Adjust the interface name(s) `igb0` to correspond with your hardware, the interface below is referenced as igb0 with a corresponding alias `WAN`, It is also possible to add a friendly name below 
    
    Add/remove sections, depending on the number of interfaces you have.

        ### Change interface as desired ###
        if [interface][name] =~ /^igb0$/ {
        mutate {
            add_field => { "[interface][alias]" => "WAN" }
            add_field => { "[network][name]" => "ISP Provider" }
            }
        }

### Forwarding pfSense Logs to Logstash

1. In pfSense navigate to Status -> System Logs -> Settings
2. General Logging Options
   - Show log entries in reverse order (newest entries on top)
3. General Logging Options > Log firewall default blocks (optional)
   - Log packets matched from the default block rules in the ruleset
   - Log packets matched from the default pass rules put in the ruleset
   - Log packets blocked by 'Block Bogon Networks' rules
   - Log packets blocked by 'Block Private Networks' rules
   - Log errors from the web server process
4. Remote Logging Options:
   - check "Send log messages to remote syslog server"
   - Select a specific interface to use for forwarding (Optional)
   - Select `IPv4` for IP Protocol
   - Enter the Logstash server local IP into the field `Remote log servers` with port 5140 (eg 192.168.1.50:5140)
   - Under "Remote Syslog Contents" check "Everything"

![pfsesne-settings](../.images/image3.png)

### Install Log Analytics Plugin

1. Run the command to install the [Azure Log Analytics](https://github.com/yokawasa/logstash-output-azure_loganalytics) plugin

    sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-azure_loganalytics

2. Configuration

        output {
            azure_loganalytics {
                customer_id => "<OMS WORKSPACE ID>"
                shared_key => "<CLIENT AUTH KEY>"
                log_type => "<LOG TYPE NAME>"
            }
        }

    Example:

        output {
            azure_loganalytics {
                customer_id => "1234567-7654321-345678-12334445"
                shared_key => "kflsdjkgfslfjsdf0ife0f0efe0-09f0we9f-ef-w00e-0w-f0w-0fwe-f0d0-w=="
                log_type => "pfsense_logstash"
            }
        }

3. Restart LogStash

        sudo systemctl restart logstash

4. Troubleshooting

        cat /var/log/logstash/logstash-plain.log

### View pfSense Logs in Azure Sentinel

1. Wait for logs to arrive in Azure Sentinel
   - This can take up to 20 minutes

   ![Azure-Sentinel](../.images/image2.png)

### Query logs in Azure Sentinel

Using the query below we can query if we are getting logs with GeoIP information into Azure Sentinel

    // PFSesne GeoIp Traffic
    pfsense_logstash_CL
    | where TimeGenerated > ago(1m)
    | where tags_s contains "GeoIP_Destination"
    | extend event_created_t = TimeGenerated
    | extend ruleName = iif(rule_uuid_s == '1572780236', 'Rule ID Name',
      iif(rule_uuid_s == '1572814498', 'Rule ID Name',
      iif(rule_uuid_s == '1572814518', 'Rule ID Name',
      iif(rule_uuid_s == '1572820907', 'Rule ID Name',
      iif(rule_uuid_s == '1572822148', 'Rule ID Name',
      iif(rule_uuid_s == '1572822176', 'Rule ID Name',
      iff(rule_uuid_s == '1572822195', 'Rule ID Name',
      iff(rule_uuid_s == '1575628181', 'Rule ID Name',
    ''))))))))
    | project TimeGenerated, interface_alias_s, network_name_s, interface_name_s, source_ip_s, source_port_s, source_geo_region_name_s, source_geo_country_iso_code_s,
        source_geo_country_name_s, destination_ip_s, destination_port_s, destination_geo_region_name_s, destination_geo_country_code3_s,
        network_direction_s, event_action_s, event_reason_s, ruleName, destination_service_s, network_transport_s
