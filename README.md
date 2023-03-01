Первым делом создаются 2 вирт машины. Для этого используем вагрант файл с конфигом.
<details>
  <summary>Конфиг</summary>  
	
```  
        MACHINES = {
     :server => {
	      :box_name => "centos/7",
	      :box_version => "2004.01",
	      :provision => "init.sh",
	      :ip => "192.168.50.10",

   },
     :client => {
        :box_name => "centos/7",
      	:box_version => "2004.01",
      	:provision => "init.sh",
      	:ip => "192.168.50.11",
   },
}


Vagrant.configure("2") do |config|

    	MACHINES.each do |boxname, boxconfig|

	      	config.vm.define boxname do |box|

	      		box.vm.box = boxconfig[:box_name]
	      		box.vm.box_version = boxconfig[:box_version]
	      		box.vm.host_name = boxname
	      		box.vm.network "private_network", ip: boxconfig[:ip]

		      	box.vm.provider :virtualbox do |vb|
		    	    	vb.customize ["modifyvm", :id, "--memory", "1024"]
		      	end

	      		box.vm.provision "shell",
                name: "configuretion_from_shell",
                path: boxconfig[:provision]
		    	end
	  	end
	end
```
</details>
