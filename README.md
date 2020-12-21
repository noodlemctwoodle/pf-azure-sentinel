# pfSense/OPNsense syslog to Azure Sentinel

This project is something I have been attempting to get working for a while now and I wanted to share it with you.

## Configuration

For Deployments please use the [Logstash Guide](Logstash-Configuration/README.md)

This project exposes the following pfSense/OPNsense data points to Azure Sentinel:

</br>

| | | | | | |
|---|---|---|---|---|---|
|TenantId|SourceSystem|TimeGenerated|log_syslog_priority_s|log_original_s|_version_s
|ecs_version_s|rule_ruleset_s|rule_uuid_s|destination_port_s|destination_geo_ip_s|destination_geo_region_name_s
|destination_geo_country_iso_code_s|destination_geo_timezone_s|destination_geo_country_code3_s|destination_geo_country_name_s|destination_geo_region_iso_code_s|destination_geo_continent_code_s
|destination_geo_longitude_d|destination_geo_latitude_d|destination_geo_location_lon_d|destination_geo_location_lat_d|destination_service_s|destination_ip_s
|source_port_s|source_geo_ip_s|source_geo_region_name_s|source_geo_country_iso_code_s|source_geo_city_name_s|source_geo_timezone_s
|source_geo_country_code3_s|source_geo_country_name_s|source_geo_postal_code_s|source_geo_region_iso_code_s|source_geo_continent_code_s|source_geo_longitude_d
|source_geo_latitude_d|source_geo_location_lon_d|source_geo_location_lat_d|source_ip_s|network_transport_s|network_name_s
|network_type_s|network_direction_s|network_iana_number_s|process_name_s|interface_name_s|interface_alias_s
|observer_name_s|observer_product_s|observer_type_s|observer_serial_number_s|pf_transport_data_length_s|pf_packet_length_s
|pf_tcp_options_s|pf_tcp_sequence_number_s|pf_tcp_flags_s|pf_ipv4_tos_s|pf_ipv4_packet_id_s|pf_ipv4_offset_s
|pf_ipv4_flags_s|tags_s - "pf" "GeoIP_Source" "GeoIP_Destination"|event_action_s|event_reason_s|event_created_t|_timestamp_t
| rule_alias_s | rule_description_s | destination_as_organization_name_s

</br>

Using the Azure Sentinel [KQL](KQL/pfSense/) we can break down this data into readable formats

![pfsense-GeoIP](.images/image1.png)

Optionally you can also install the [Linux OMS Agent](Linux-OMS-Agent/README.md) to collect performance stats from the log collector. This is useful for creating analytics rules that will monitor for disk space alerts and excessive CPU usage.

## Credits

This project is only possible with the work carried out by [a3ilson](https://github.com/pfelk/pfelk) and his pfELK project to parse the pfSense log files.
