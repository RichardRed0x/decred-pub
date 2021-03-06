

Create a VPS 

SSH into it.

## Decred Installation

First lets go to your home folder:

	cd ~

Get latest version of dcrinstall

    wget https://github.com/decred/decred-release/releases/download/v1.6.0-rc1/dcrinstall-linux-amd64-v1.6.0-rc1

change permissions to executable:

	chmod +x dcrinstall-linux-amd64-v1.6.0-rc1

execute dcrinstall

	./dcrinstall-linux-amd64-v1.6.0-rc1

this will download dcrd, dcrwallet and dcrlnd - as part of the process it will also create a new wallet within dcrwallet and generate the seed words, write these down and store somewhere safe, you will get the following prompt:

	Once you have stored the seed in a safe and secure location, enter "OK" to continue:

dcrd, dcrwallet and dcrlnd have now been installed.

## Decred Full Node
### Configuration

The dcrd full node needs to have the **txindex** option enabled, so modify the dcrd.conf by running the following command:

	nano ~/.dcrd/dcrd.conf

The configuration file has a lot of settings, scroll down to **Optional Indexes** section and uncomment (remove the ;) following setting:
   
    txindex=1

Save the configuration file by presssing **CTRL + X**, then **Y**, then **Enter**.

### Running dcrd

I like to use tmux to have different consoles within a single ssh session, you can create a new tmux session for dcrd with the following command. This step is not required if you have desktop access, you can just create a new terminal instead.

	tmux new -s dcrd

this will give you a brand new terminal that you can check when needed, now go to the decred folder that was created by dcrinstall

	cd ~/decred

and start up dcrd:

	./dcrd

At this point the decred blockchain will start downloading and the node will be synchronising, this process usually takes a few hours depending on the hardware/broadband speed.

You can detach from the tmux session by pressing **CTRL+B**, then **D**. If you need to check back in to see the status of the node you can reattach the session by using the following command:

	tmux attach-session -t dcrd

I like to create an alias to simplify this:

	alias tmux_dcrd="tmux attach-session -t dcrd"

Afterwards, you should be able to attach the session by just entering the following command:

	tmux_dcrd

As before, get back to your main terminal by pressing **CTRL+B**, then **D**.

To know if your node has fully synched, look at the 'height' in the last message in the dcrd session and compare against the latest block on dcrdata.org.

## Decred Lightning Network Daemon 

### Configuration

Settings for your dcrlnd node can be modified by opening up the dcrlnd.conf with the following command:

	nano ~/.dcrlnd/dcrlnd.conf

In my case, I'm want this node to be public and it will be running off of a VPS with a static ip address, and I want to make it public, so I can modify the following setting:

	; Adding an external IP will advertise your node to the network. This signals
	; that your node is available to accept incoming channels. If you don't wish to
	; advertise your node, this value doesn't need to be set. Unless specified
	; (with host:port notation), the default port (9735) will be added to the
	; address.
	 externalip=<enter your address here>

You can also modify further down the alias of the node and the colour that it will be displayed in.

If your intention is to use Ride The Lightning, you will also need to enable listening for gRPC connections:


### Running

Create a new tmux session or open a new terminal for dcrlnd
	 
	 tmux new -s dcrlnd

now go to the decred folder that was created by dcrinstall

	cd ~/decred

and start up dcrd:

	./dcrlnd

You will be prompted to create and unlock the wallet:

	2020-10-24 09:09:11.188 [INF] LTND: Waiting for wallet encryption password. Use `dcrlncli create` to create a wallet, `dcrlncli unlock` to unlock an existing wallet, or `dcrlncli changepassword` to change the password of an existing wallet and unlock it.

Detach from the session by pressing **CTRL+B**, then **D**. 

The next step is to use dcrlncli to create a wallet, this will be your main wallet for funding lightning channels and is different from the dcrwallet configured during installation,  I cannot stress enoguh that you have to **write down the seed**.

	~/decred/dcrlncli create

For those of you using tmux, you can create an alias to call the dcrlnd session:

	alias tmux_dcrlnd="tmux attach -t dcrlnd"

Go back to the dcrlnd terminal, and you will now see it updating and synchronising. Once fully synchronised you'll be ready to start funding the wallet and opening channels.

### Command line usage

At this stage you're basically set to start

You can generate a new address by using the command below:

	~/decred/dcrlncli newaddress

check your balance:

	~/decred/dcrlncli walletbalance

and send coins to another wallet:

	~/decred/dcrlncli sendcoins <address> <amount>

connect to an online node:

	~/decred/dcrlncli connect <node address>

## Ride the Lightning

An obvious disclaimer here, RTL is developed for Bitcoin. However given that dcrlnd is a fork of upstream lnd and uses the same BOLTs and APIs, a lot of the functionality *should work*. At the time of writing I've tested the following:

* Opening channels ✅
* Adding peers ✅
* Closing channels ✅
* Receiving mainnet coins - this probably will need tweaking as RTL has bech32 option and you can't disable it.
* Sending mainnet coins
*  Sending LN payments
*  Creating LN invoice
 

### Installation

First lets take of the prerequisites, starting by [node.js](https://nodejs.org/en/download/). The instance I'm using is based on ubuntu, in this case the command is:

	curl -sL https://deb.nodesource.com/setup_15.x | sudo 	-E bash -

and then:

	sudo apt-get install -y nodejs

We also need to ensure we have g++:

	sudo apt-get install -y build-essential

Once completed we do a pull from the RTL repository:

	cd ~
	git clone https://github.com/Ride-The-Lightning/RTL.git

Then browse into the RTL folder and install:
	
	cd RTL
	npm install --only=prod

This part takes a while... Once it's complete we're ready to modify the configuration file before starting it up.

### Configuration

RTL includes a sample configuration, I've modified it to work with the setup we've below - **read a bit further down instead of just copying and pasting**:

    {
      "multiPass": "password",
      "port": "3000",
      "defaultNodeIndex": 1,
      "SSO": {
        "rtlSSO": 0,
        "rtlCookiePath": "",
        "logoutRedirectLink": ""
      },
      "nodes": [
        {
          "index": 1,
          "lnNode": "My Node",
          "lnImplementation": "LND",
          "Authentication": {
            "macaroonPath": "/home/user/.dcrlnd/data/chain/decred/mainnet",
            "configPath": "/home/user/.dcrlnd/dcrlnd.conf"
          },
          "Settings": {
            "userPersona": "OPERATOR",
            "themeMode": "DAY",
            "themeColor": "PURPLE",
            "channelBackupPath": "~/.dcrlnd/data/chain/decred/mainnet",
            "enableLogging": false,
            "lnServerUrl": "https://localhost:8080",
            "swapServerUrl": "http://localhost:8081",
            "fiatConversion": false
          }
        }
      ]
    }

Key parts that have changed from the sample configuration:

The password can stay unchanged initially, you will get prompted to change it and will be encrypted as soon as you login the first time.

The port is 3000 by default, if your node is public you may want to consider changing this as its the port used to access the web interface:

	"port": "3000",

The macaroon and config paths are standard given the dcrlnd configuration that we just deployed with this installation method:

	"macaroonPath": "/home/user/.dcrlnd/data/chain/decred/mainnet"
	"configPath": "/home/user/.dcrlnd/dcrlnd.conf"

User mode changes some of the interface, I'm assuming the majority of people using this at this stage prefer the node operator interface than the merchant one:

	"userPersona": "OPERATOR",

The channel backup path is also standard with the setup that we've used for this tutorial:
	           
	"channelBackupPath": "/home/user/.dcrlnd/data/chain/decred/mainnet",

You can create the new config file by doing

	nano ~/RTL/RTL-Config.json
		
and pasting the modified config in, then save by pressing **CTRL+X**, then **Y**, then **ENTER**.

At this stage, open a new terminal or create a new tmux session:

	tmux new -s rtl

and run RTL:

	node rtl

You should get the following message:

	Please note that, RTL has encrypted the plaintext password into its corresponding hash.Server is up and running, please open the UI at http://localhost:3000

If you open a browser window and go to the ip address of the node with the specified port you'll be greeted with the RTL password prompt window, and asked to change it.
