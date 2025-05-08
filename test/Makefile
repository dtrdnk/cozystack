.DEFAULT_GOAL := all
VM_ORDER_NUMBERS = 1 2 3
BRIDGE_INTERFACE=cozy-br0
BRIDGE_IP=192.168.123.1/24
VM_LIST = $(foreach suffix,$(VM_ORDER_NUMBERS),srv$(suffix))
APID_PORT = 50000
TIMEOUT = 60
RETRY_INTERVAL = 1
export KUBECONFIG := ./kubeconfig
export TALOSCONFIG := ./talosconfig

.PHONY: all
all: check-installer-env check-ip-forwarding clean \
	prepare-host prepare-servers run-servers check-ports \
	prepare-talos

.PHONY: check-installer-env
check-installer-env:
ifndef COZYSTACK_INSTALLER_YAML
	@echo 'COZYSTACK_INSTALLER_YAML variable is not set!' >&2
	@echo 'please set it with following command:' >&2
	@echo >&2
	@echo 'export COZYSTACK_INSTALLER_YAML=$$(helm template -n cozy-system installer packages/core/installer)' >&2
	@echo >&2
	exit 1
endif

.PHONY: check-ip-forwarding
check-ip-forwarding:
ifneq ($(file < /proc/sys/net/ipv4/ip_forward), 1)
	@echo "IPv4 forwarding is not enabled!" >&2
	@echo 'please enable forwarding with the following command:' >&2
	@echo >&2
	@echo 'echo 1 > /proc/sys/net/ipv4/ip_forward' >&2
	@echo >&2
	exit 1
endif

.PHONY:
prepare-host: create-bridge setup-nat

create-bridge:
	@echo "Creating bridge interface $(BRIDGE_INTERFACE)"
	@if ! ip link show $(BRIDGE_INTERFACE) >/dev/null 2>&1; then \
		ip link add $(BRIDGE_INTERFACE) type bridge; \
		ip link set $(BRIDGE_INTERFACE) up; \
		ip addr add $(BRIDGE_IP) dev $(BRIDGE_INTERFACE); \
		echo "Bridge interface $(BRIDGE_INTERFACE) with $(BRIDGE_IP) created"; \
	else \
	  echo "Bridge $(BRIDGE_INTERFACE) already exists"; \
	fi
	@echo ""

delete-bridge:
	@if ip link show $(BRIDGE_INTERFACE) >/dev/null 2>&1; then \
  		ip link set $(BRIDGE_INTERFACE) down; \
  		ip link del $(BRIDGE_INTERFACE); \
  		echo "Bridge interface $(BRIDGE_INTERFACE) deleted"; \
  	else \
  	  echo "Bridge $(BRIDGE_INTERFACE) does not exist"; \
  	fi
	@echo ""

setup-nat:
	@echo "Setting up NAT masquerading on host"
	iptables -t nat -A POSTROUTING -s 192.168.123.0/24 ! -d 192.168.123.0/24 -j MASQUERADE 2>/dev/null || true
	@echo ""

clear-nat:
	@echo "Remove NAT masquerading rule on host"
	iptables -t nat -D POSTROUTING -s 192.168.123.0/24 ! -d 192.168.123.0/24 -j MASQUERADE 2>/dev/null || true
	@echo ""

.PHONY:
prepare-servers: download-nocloud-raw create-dirs create-system-disks \
				create-data-disks generate-meta-data generate-network-config \
				generate-user-data create-seed-disks prepare-servers-network
	@echo "Preparing servers finished! Ignition!"

download-nocloud-raw:
ifeq (,$(wildcard ./nocloud-amd64.raw))
	curl -L https://github.com/cozystack/cozystack/releases/latest/download/nocloud-amd64.raw.xz \
		-o nocloud-amd64.raw.xz
	# TODO: recreate nocloud-amd64.raw for fresh installations
	# rm -rf nocloud-amd64.raw.xz
	xz --decompress nocloud-amd64.raw.xz
endif

$(VM_LIST):
	@echo "create directory $@"
	@mkdir -p $@

create-dirs: $(VM_LIST)
	@echo ""

.PHONY: clean-dirs
clean-dirs:
	@echo "Cleaning server directories"
	@rm -rf $(VM_LIST);
	@echo ""

create-system-disks: download-nocloud-raw
	@$(foreach vm,$(VM_LIST), \
	echo "Resizing system.img for $(vm)"; \
	cp nocloud-amd64.raw $(vm)/system.img; \
  	qemu-img resize $(vm)/system.img 20G; \
	)
	@echo ""

.PHONY: create-data-disks
create-data-disks:
	@$(foreach vm,$(VM_LIST), \
	echo "Creating data disks for $(vm)"; \
	qemu-img create $(vm)/data.img 100G;)
	@echo ""

.PHONY: create-seed-disks
create-seed-disks:
	@echo "Creating servers seed disks";
	@$(foreach vm,$(VM_LIST), \
	echo "Creating seed disk for $(vm)"; \
	$(shell cd $(vm) && genisoimage -output seed.img \
	-volid cidata -rational-rock -joliet user-data meta-data network-config))
	@echo ""

generate-meta-data:
	@echo "Generating meta-data files"
	@$(foreach vm,$(VM_LIST), \
        echo "hostname: $$(basename $(vm))" > $(vm)/meta-data; \
        )
	@echo ""

generate-user-data:
	@echo "Generating user-data files"
	@$(foreach vm,$(VM_LIST), \
        echo "#cloud-config" > $(vm)/user-data; \
        )
	@echo ""

generate-network-config:
	@echo "Creating network-config files"
	@$(foreach vm,$(VM_ORDER_NUMBERS), \
		sed -e "s/PLACEHOLDER_NAME/192.168.123.1$(vm)\/26/" \
		network_config_template.yaml > srv$(vm)/network-config;)
	@echo ""

.PHONY: run-servers
run-servers:
	@$(foreach vm,$(VM_ORDER_NUMBERS), \
		echo "Running server srv$(vm) in QEMU"; \
          qemu-system-x86_64 -machine type=pc,accel=kvm -cpu host -smp 8 -m 16384 \
            -device virtio-net,netdev=net0,mac=52:54:00:12:34:5$(vm) \
            -netdev tap,id=net0,ifname=cozy-srv$(vm),script=no,downscript=no \
            -drive file=srv$(vm)/system.img,if=virtio,format=raw \
            -drive file=srv$(vm)/seed.img,if=virtio,format=raw \
            -drive file=srv$(vm)/data.img,if=virtio,format=raw \
            -display none -daemonize -pidfile srv$(vm)/qemu.pid; \
      )
	@echo ""

.PHONY: stop-servers
stop-servers:
	@$(foreach vm, $(VM_LIST), \
		if [ -e $(vm)/qemu.pid ]; then \
		  echo "Stopping $(vm)"; \
		  kill $$(cat $(vm)/qemu.pid) || true; \
		fi;)
	@echo ""

define CHECK_PORT
	@echo "Checking if $(2) is open on 192.168.123.1$(1)"
	@timeout $(TIMEOUT) sh -c \
		'until nc -nzv 192.168.123.1$(1) $(2); do \
        echo "Waiting for 192.168.123.1$(1):$(2)..."; \
        sleep $(RETRY_INTERVAL); \
    done; \
    echo "192.168.123.1$(1):$(2) is open."'
endef

# Loop over servers and check the port
check-ports:
	$(foreach vm, $(VM_ORDER_NUMBERS), $(call CHECK_PORT,$(vm),$(APID_PORT)))
	@echo ""

.PHONY:
prepare-talos: secrets.yaml patch.yaml patch-controlplane.yaml \
	talosconfig apply-talos-configs bootstrap-talos wait-for-etcd kubeconfig

patch.yaml:
	cp patch-template.yaml patch.yaml

patch-controlplane.yaml:
	cp patch-controlplane-template.yaml patch-controlplane.yaml

talosconfig:
	talosctl gen config --with-secrets secrets.yaml \
	cozystack https://192.168.123.10:6443 \
	--config-patch=@patch.yaml --config-patch-control-plane @patch-controlplane.yaml

.PHONY: apply-talos-configs
apply-talos-configs:
	$(foreach vm, $(VM_ORDER_NUMBERS), \
	talosctl apply -f controlplane.yaml -n 192.168.123.1$(vm) -e 192.168.123.1$(vm) -i; \
	)
	@echo ""

bootstrap-talos: check-ports
#	timeout 10 sh -c 'until talosctl bootstrap -n 192.168.123.11 -e 192.168.123.11; do sleep 1; done'
	@echo "Bootstrapping Talos"
	@timeout $(TIMEOUT) sh -c \
		'until talosctl bootstrap -n 192.168.123.11 -e 192.168.123.11; do sleep 1; done'
	@echo ""

wait-for-etcd:
	@echo "Waiting for etcd..."
	@timeout 180 sh -c \
		'until timeout -s 9 2 talosctl etcd members -n 192.168.123.11,192.168.123.12,192.168.123.13 -e 192.168.123.10 2>&1; do sleep 1; done'
	@echo ""
	@timeout $(TIMEOUT) sh -c \
		'while talosctl etcd members -n 192.168.123.11,192.168.123.12,192.168.123.13 -e 192.168.123.10 2>&1 | grep "rpc error"; do sleep 1; done'
	@echo ""

secrets.yaml:
	talosctl gen secrets

kubeconfig:
	talosctl kubeconfig kubeconfig -e 192.168.123.10 -n 192.168.123.10

define REMOVE_FILES
	rm -f $(1)
endef

.PHONY: clean-files
clean-files:
	$(call REMOVE_FILES,secrets.yaml patch.yaml \
	patch-controlplane.yaml controlplane.yaml worker.yaml talosconfig kubeconfig)

.PHONY: prepare-servers-network
prepare-servers-network:
	@$(foreach vm, $(VM_LIST), \
		ip tuntap add dev cozy-$(vm) mode tap; \
        ip link set cozy-$(vm) up; \
        ip link set cozy-$(vm) master $(BRIDGE_INTERFACE); \
	)

.PHONY: clean-servers-interfaces
clean-servers-interfaces:
# TODO: add output for each removing interface
	@echo "Removing servers tap interfaces if they exists"
	@$(foreach vm, $(VM_LIST), \
  		if [ -d /sys/class/net/cozy-$(vm) ]; then \
  		  ip link del cozy-$(vm); \
  		fi;)

clean: stop-servers clean-dirs clean-files delete-bridge clear-nat clean-servers-interfaces
	@echo ""
