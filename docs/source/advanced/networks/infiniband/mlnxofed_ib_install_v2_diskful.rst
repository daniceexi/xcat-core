Configuration for Diskful Installation
=======================================

1. Set script ``mlnxofed_ib_install`` as ``postbootscripts`` or ``postscripts`` ::

	chdef <node> -p postbootscripts="mlnxofed_ib_install -p /install/<subpath>/<MLNX_OFED_LINUX.iso>" 
  
  Or ::

        chdef <node> -p postscripts="mlnxofed_ib_install -p /install/<subpath>/<MLNX_OFED_LINUX.iso>"

  xCAT simulates completely the way Mellanox scripts work by using ``postbootscripts``. This way need to reboot after drive installation to make Mellanox drivers work reliably just like Mellanox suggested. If you want to use the reboot after operating system installation to avoid reboot twice, you can using ``postscripts`` attribute to install Mellanox drivers. This way has been verified in limited scenarios. For more information please refer to :doc:`The Scenarios Have Been Verified </advanced/networks/infiniband/mlnxofed_ib_verified_scenario_matrix>`. You can try this way in other else scenarios if you needed.  
	
2. Specify dependency package

  Some dependencies need to be installed before running Mellanox scripts. These dependencies are different between different scenario. xCAT configurates these dependency packages by using ``pkglist`` attribute of ``osimage`` definition. Please refer to :doc:`Add Additional Software Packages </guides/admin-guides/manage_clusters/ppc64le/diskful/customize_image/additional_pkg>` for more information::

    # lsdef -t osimage <os>-<arch>-install-compute
    Object name: <os>-<arch>-install-compute
    imagetype=linux
    ....
    pkgdir=/<os packages directory>
    pkglist=/<os packages list directory>/compute.<os>.<arch>.pkglist
    ....

  You can append the ib dependency packages list in the end of ``/<os packages list directory>/compute.<os>.<arch>.pkglist`` directly like below: ::

    #cat /<os packages list directory>/compute.<os>.<arch>.pkglist
    @base
    @x11
    openssl
    ntp
    rsyn 
    #ib part
    createrepo
    kernel-devel
    kernel-source
    ....


  Or if you want to isolate InfiniBand dependency packages list into a separate file, after you edit this file, you can append the file in ``/<os packages list directory>/compute.<os>.<arch>.pkglist`` like below way: ::

    #cat /<os packages list directory>/compute.<os>.<arch>.pkglist
    @base
    @x11
    openssl
    ntp
    rsyn
    #INCLUDE:/<ib pkglist path>/<you ib pkglist file>#

  xCAT has shipped some ib pkglist files under ``/opt/xcat/share/xcat/ib/netboot/<ostype>/``, these pkglist files have been verified in sepecific scenario. Please refer to :doc:`The Scenarios Have Been Verified </advanced/networks/infiniband/mlnxofed_ib_verified_scenario_matrix>` to judge if you can use it directly in your environment. If so, you can use it like below: ::

    #cat /<os packages list directory>/compute.<os>.<arch>.pkglist
    @base
    @x11
    openssl
    ntp
    rsyn
    #INCLUDE:/opt/xcat/share/xcat/ib/netboot/<ostype>/ib.<os>.<arch>.pkglist#
    
  Take rhels7.2 on ppc64le for example:   ::

     # lsdef  -t osimage rhels7.2-ppc64le-install-compute
     Object name: rhels7.2-ppc64le-install-compute
     imagetype=linux
     osarch=ppc64le
     osdistroname=rhels7.2-ppc64le
     osname=Linux
     osvers=rhels7.2
     otherpkgdir=/install/post/otherpkgs/rhels7.2/ppc64le
     pkgdir=/install/rhels7.2/ppc64le
     pkglist=/install/custom/install/rh/compute.rhels7.ib.pkglist
     profile=compute
     provmethod=install
     template=/opt/xcat/share/xcat/install/rh/compute.rhels7.tmpl
		

  **[Note]**: If the osimage definition was generated by xCAT command ``copycds``, default value ``/opt/xcat/share/xcat/install/rh/compute.rhels7.pkglist`` was assigned to ``pkglist`` attribute. ``/opt/xcat/share/xcat/install/rh/compute.rhels7.pkglist`` is the sample pkglist shipped by xCAT, recommend to make a copy of this sample and using the copy in real environment. In the above example, ``/install/custom/install/rh/compute.rhels7.ib.pkglist`` is a copy of ``/opt/xcat/share/xcat/install/rh/compute.rhels7.pkglist``. ::

    # cat /install/custom/install/rh/compute.rhels7.ib.pkglist
    #Please make sure there is a space between @ and group name
    wget
    ntp
    nfs-utils
    net-snmp
    rsync
    yp-tools
    openssh-server
    util-linux
    net-tools
    #INCLUDE:/opt/xcat/share/xcat/ib/netboot/rh/ib.rhels7.ppc64le.pkglist#


 
3. Install node ::

	nodeset <node> osimage=<osver>-<arch>-install-compute
	rsetboot <node> net
	rpower <node> reset


  After steps above, you can login target node and find the Mellanox InfiniBand drives are located under ``/lib/modules/<kernel_version>/extra/``. 

  Issue ``ibv_devinfo`` command you can get the InfiniBand apater information ::

    # ibv_devinfo
    hca_id:	mlx5_0
	transport:			InfiniBand (0)
	fw_ver:				10.14.2036
	node_guid:			f452:1403:0076:10e0
	sys_image_guid:			f452:1403:0076:10e0
	vendor_id:			0x02c9
	vendor_part_id:			4113
	hw_ver:				0x0
	board_id:			IBM1210111019
	phys_port_cnt:			2
	Device ports:
		port:	1
			state:			PORT_INIT (2)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			0
			port_lid:		65535
			port_lmc:		0x00
			link_layer:		InfiniBand

		port:	2
			state:			PORT_DOWN (1)
			max_mtu:		4096 (5)
			active_mtu:		4096 (5)
			sm_lid:			0
			port_lid:		65535
			port_lmc:		0x00
			link_layer:		InfiniBand 

  Using ``service openibd status`` to verify if openibd works well. Below is the output in rhels7.2. ::


    # service openibd status
      HCA driver loaded
    
    Configured IPoIB devices:
    ib0 ib1
    
    Currently active IPoIB devices:
    Configured Mellanox EN devices:
    
    Currently active Mellanox devices:
    
    The following OFED modules are loaded:
    
      rdma_ucm
      rdma_cm
      ib_addr
      ib_ipoib
      mlx4_core
      mlx4_ib
      mlx4_en
      mlx5_core
      mlx5_ib
      ib_uverbs
      ib_umad
      ib_ucm
      ib_sa
      ib_cm
      ib_mad
      ib_core


