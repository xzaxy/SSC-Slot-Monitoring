# Monitoring SlotsBehind for your Solana RPC node with Grafana and Prometheus


This guide aims to help export your node's slotsBehind stats in near real-time so you can monitor with Grafana.

#
## Step 1 - Create a directory for the nodeexperter Textfile Collector to read

    sudo mkdir /var/lib/node_exporter/textfile_collector

#
## Step 2 - Create the cron jobs to schedule the stats export

Add a `cron job` to place the stats into text files in that directory for the `Textfile Collector`, in a format fit for `nodeExporter`:

**To monitor slot status:**

    sudo nano /etc/cron.d/solana_slots_stats

In the "solana_slots_stats" file paste the following  (`*/2` means I have mine updating every 2 minutes):

    */2 * * * * root curl http://localhost:8899 -k -X POST -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1, "method":"getHealth"}' 2>&1 | sed -nr 's/^(.*)(numSlotsBehind":)([0-9]+)(.*)$/node_solana_slots_status{Stats="SlotsBehind"} \3/p' > /var/lib/node_exporter/textfile_collector/solana_slots_stats.prom.$$ && mv /var/lib/node_exporter/textfile_collector/solana_slots_stats.prom.$$ /var/lib/node_exporter/textfile_collector/solana_slots_stats.prom

**Bonus - To monitor directory sizes:**
	
    sudo nano /etc/cron.d/directory_size
		
In the `directory_size` file paste the following  (`*/5` means I have mine updating every 5 minutes, and I'm monitoring the sizes of three directories; `/mt/accounts`, `/mt/ledger/validator-ledger/rocksdb` and `/mt/ledger/validator-ledger/accounts_index` ):

    */5 * * * * root du -sb /mt/accounts /mt/ledger/validator-ledger/rocksdb /mt/ledger/validator-ledger/accounts_index | sed -ne 's/^\([0-9]\+\)\t\(.*\)$/node_directory_size_bytes{directory="\2"} \1/p' > /var/lib/node_exporter/textfile_collector/directory_size.prom.$$ && mv /var/lib/node_exporter/textfile_collector/directory_size.prom.$$ /var/lib/node_exporter/textfile_collector/directory_size.prom`

#
## Step 3 - Modify the nodeExporter config in docker-compose.yml:

    sudo nano docker-compose.yml

Under the "nodeexperter" section in the docker-compose.yml file, add the directory we created earlier to the container's mapped volumes:

    - /var/lib/node_exporter/textfile_collector/:/host/textfile_collector:ro
	 
Also add the command for nodeExporter to run the Textfile Collector on that directory:

    - '--collector.textfile.directory=/host/textfile_collector'
	 
	 
e.g. docker-compse.yml file snippet:
```
	[...]
	nodeexporter:
		image: prom/node-exporter
		container_name: nodeexporter
		volumes:
		  - /proc:/host/proc:ro
		  - /sys:/host/sys:ro
		  - /var/lib/node_exporter/textfile_collector/:/host/textfile_collector:ro
		  - /:/rootfs:ro
		command:
		  - '--path.procfs=/host/proc'
		  - '--path.rootfs=/rootfs'
		  - '--path.sysfs=/host/sys'
		  - '--collector.textfile.directory=/host/textfile_collector'
		  - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
	[...]
```
Once you've made the changes to the docker-comse.yml file, change  directory to where your docker-compose.yml resides and restart the containers to make the changes take effect (e.g. `cd ~/shadow-monitoring`):

	sudo docker-compose up -d

#
## Step 4 - Check in Grafana that the stats have made it to Prometheus 
In Grafana, after 5 or so minutes we can verify that the stats are visible.

Open Explorer, and begin typing `node_sol`, if everything is working, you should see the new node name we're collecting autofill:

![image](https://user-images.githubusercontent.com/21113750/171871603-fa6e8ad5-bc47-4dd8-9a86-2d301bbe03bf.png)

Select it, and type `{` and click "Stats":

![image](https://user-images.githubusercontent.com/21113750/171871628-2f7ab1b4-d4d4-492d-8f43-9e83bcef30e9.png)

Then click "SlotsBehind":
	
![image](https://user-images.githubusercontent.com/21113750/171872237-3ed4d5b4-d8e8-4180-a1e1-59501d52812f.png)
	
Now run the query:
	
![image](https://user-images.githubusercontent.com/21113750/171872257-fe89729b-8d23-491f-92bf-c17846b0b858.png)
	
If it's working (and your node is behind) you should now see the graph:

![image](https://user-images.githubusercontent.com/21113750/171872385-2e41128a-cf00-42d7-a585-d901c45a2b5b.png)

You can now add this query to your dashboard!
