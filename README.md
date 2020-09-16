# Deploying your application with NixOS

We'll see how one can leverage the Nix ecosystem to easily deploy web applications in a declarative and reproducible way.
We'll demonstrate how to get NixOS running on most servers, even when NixOS is not officially supported by the hosting provider.
The application we are about to deploy is a simple Haskell app which requires access to a PostgreSQL database and also a Redis instance, as an excuse to demonstrate how to orchestrate the different building blocks of a deployment.
Let's begin !

## Getting NixOS on a VPS

Here, we'll use a simple VPS from Hetzner Cloud but the following should work with most providers.
I won't demonstrate the creation of the VPS as it's provider specific and often just requires a few clicks.
We'll start from a Debian installation which I picked from the available choices offered by Hetzner when creating a new server.
I'm now logged in as root in my new VPS, this will be our "blank state":

### The NixOS Lustrate method
~~~bash
Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Jun 11 19:27:48 2020 from 37.166.144.196

root@debian-2gb-nbg1-1:~# uname -a
Linux debian-2gb-nbg1-1 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1+deb10u1 (2020-04-27) x86_64 GNU/Linux
~~~

What we want to achieve now is replacing the current Debian installation with a NixOS one. To do this we'll be using the **_NIXOS_LUSTRATE_** method as explained in the [section of the NixOS manual](https://nixos.org/nixos/manual/#sec-installing-from-other-distro).  
First a quick look at the current disk partitionning:
~~~bash
root@debian-2gb-nbg1-1:~# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0 19.1G  0 disk
├─sda1    8:1    0   19G  0 part /
├─sda14   8:14   0    1M  0 part
└─sda15   8:15   0  122M  0 part /boot/efi
sr0      11:0    1 1024M  0 rom
~~~

Let's add a user to run all the installation steps as running it as root triggers some warnings:
~~~bash
root@debian-2gb-nbg1-1:~# adduser romain
Adding user `romain' ...
Adding new group `romain' (1000) ...
Adding new user `romain' (1000) with group `romain' ...
Creating home directory `/home/romain' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for romain
Enter the new value, or press ENTER for the default
	Full Name []:
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n]
~~~
We then add our new user to the sudo group to be able to proceed with the next steps:
~~~bash
root@debian-2gb-nbg1-1:~# usermod -aG sudo romain
~~~
We switch to the new user:
~~~bash
su - romain
~~~
Now we can begin the nix installation process as usual:
~~~bash
curl https://nixos.org/nix/install | sh
. $HOME/.nix-profile/etc/profile.d/nix.sh
~~~
Once this is done, we can see that we are currently subscribed to the **nixpkgs-unstable** channel:
~~~bash
nix-channel --list
nixpkgs https://nixos.org/channels/nixpkgs-unstable
~~~
We should probably use a stable channel on a server, so let's replace **nixpkgs-unstable** with the latest nixos channel. That's **nixos-20.03** at the time I write this blog post.
~~~bash
nix-channel --add https://nixos.org/channels/nixos-20.03 nixpkgs
~~~
We then update just to be sure we are using the latest sources:
~~~bash
nix-channel --update
~~~
It's time to install the NixOS installation utilities **nixos-generate-config**, **nixos-install** and **nixos-enter**:
~~~bash
nix-env -iE "_: with import <nixpkgs/nixos> { configuration = {}; }; with config.system.build; [ nixos-generate-config nixos-install nixos-enter ]"
~~~
Once it's done, we have to generate NixOS configuration files.
~~~bash
romain@debian-2gb-nbg1-1:~$ sudo `which nixos-generate-config` --root /
writing /etc/nixos/hardware-configuration.nix...
writing /etc/nixos/configuration.nix...
~~~
You may have encountered the following warnings when invoking the previous command (I have).
~~~bash
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
~~~
This doesn't seem to have any noticeable impact down the line so let's proceed.

Let's now take a look at those two files **nixos-generate-config** created for us.

**/etc/nixos/configuration.nix**
~~~nix
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).

{ config, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];

  # Use the GRUB 2 boot loader.
  boot.loader.grub.enable = true;
  boot.loader.grub.version = 2;
  # boot.loader.grub.efiSupport = true;
  # boot.loader.grub.efiInstallAsRemovable = true;
  # boot.loader.efi.efiSysMountPoint = "/boot/efi";
  # Define on which hard drive you want to install Grub.
  # boot.loader.grub.device = "/dev/sda"; # or "nodev" for efi only

  # networking.hostName = "nixos"; # Define your hostname.
  # networking.wireless.enable = true;  # Enables wireless support via wpa_supplicant.

  # The global useDHCP flag is deprecated, therefore explicitly set to false here.
  # Per-interface useDHCP will be mandatory in the future, so this generated config
  # replicates the default behaviour.
  networking.useDHCP = false;
  networking.interfaces.eth0.useDHCP = true;

  # Configure network proxy if necessary
  # networking.proxy.default = "http://user:password@proxy:port/";
  # networking.proxy.noProxy = "127.0.0.1,localhost,internal.domain";

  # Select internationalisation properties.
  # i18n.defaultLocale = "en_US.UTF-8";
  # console = {
  #   font = "Lat2-Terminus16";
  #   keyMap = "us";
  # };

  # Set your time zone.
  # time.timeZone = "Europe/Amsterdam";

  # List packages installed in system profile. To search, run:
  # $ nix search wget
  # environment.systemPackages = with pkgs; [
  #   wget vim
  # ];

  # Some programs need SUID wrappers, can be configured further or are
  # started in user sessions.
  # programs.mtr.enable = true;
  # programs.gnupg.agent = {
  #   enable = true;
  #   enableSSHSupport = true;
  #   pinentryFlavor = "gnome3";
  # };

  # List services that you want to enable:

  # Enable the OpenSSH daemon.
  # services.openssh.enable = true;

  # Open ports in the firewall.
  # networking.firewall.allowedTCPPorts = [ ... ];
  # networking.firewall.allowedUDPPorts = [ ... ];
  # Or disable the firewall altogether.
  # networking.firewall.enable = false;

  # Enable CUPS to print documents.
  # services.printing.enable = true;

  # Enable sound.
  # sound.enable = true;
  # hardware.pulseaudio.enable = true;

  # Enable the X11 windowing system.
  # services.xserver.enable = true;
  # services.xserver.layout = "us";
  # services.xserver.xkbOptions = "eurosign:e";

  # Enable touchpad support.
  # services.xserver.libinput.enable = true;

  # Enable the KDE Desktop Environment.
  # services.xserver.displayManager.sddm.enable = true;
  # services.xserver.desktopManager.plasma5.enable = true;

  # Define a user account. Don't forget to set a password with ‘passwd’.
  # users.users.jane = {
  #   isNormalUser = true;
  #   extraGroups = [ "wheel" ]; # Enable ‘sudo’ for the user.
  # };

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "20.03"; # Did you read the comment?

}
~~~

This is your **logical** configuration where you can specify what you want to include in your system regardless of the physical machine you build it on.

**/etc/nixos/hardware-configuration.nix**
~~~nix
# Do not modify this file!  It was generated by ‘nixos-generate-config’
# and may be overwritten by future invocations.  Please make changes
# to /etc/nixos/configuration.nix instead.
{ config, lib, pkgs, ... }:

{
  imports =
    [ <nixpkgs/nixos/modules/profiles/qemu-guest.nix>
    ];

  boot.initrd.availableKernelModules = [ "ata_piix" "virtio_pci" "xhci_pci" "sd_mod" "sr_mod" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ ];
  boot.extraModulePackages = [ ];

  fileSystems."/" =
    { device = "/dev/disk/by-uuid/c97a4f38-9c51-4846-af75-775ac08e6a76";
      fsType = "ext4";
    };

  fileSystems."/boot/efi" =
    { device = "/dev/disk/by-uuid/3143-1D20";
      fsType = "vfat";
    };

  swapDevices = [ ];

  nix.maxJobs = lib.mkDefault 1;
}

~~~
This is your **physical** configuration which NixOS probed from the current machine.  
You can see that it set **nix.maxJobs** to default to 1 as my VPS only has one core.  
If you want to learn about all the options available to configure your system, you'll want to browse them here [NixOS options](https://nixos.org/nixos/options#).  
For example, the docs tell us the following about **nix.maxJobs**:  
_This option defines the maximum number of jobs that Nix will try to build in parallel. The default is 1. You should generally set it to the total number of logical cores in your system (e.g., 16 for two CPUs with 4 cores each and hyper-threading)._


Right, we'll trim down our **configuration.nix** by removing comments and adding needed ssh and network configuration.
~~~nix
{ config, pkgs, ... }:

{

  ...
  
  networking.usePredictableInterfaceNames = false;

  ...
  
  services.openssh.enable = true;

  # In case you want to be able to login as root without public key:
  # services.openssh.permitRootLogin = "yes";
  # nix-shell -p mkpasswd --run 'mkpasswd -m sha-512 nixosdemo'
  # users.users.root.initialHashedPassword = "$6$ElZ9YNBW/hBJYjo$7TSbGs.H0abwIHUJe3zPQ8NScs6AKOKB7Br6TBEZk.vgZ7J9neAVL8CbTIGOPfW8oirGP1b6kRErQvo/r9jmX1";
  
  users.users.root.openssh.authorizedKeys.keys = [
    "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICRH41b8Qn+80OJuZssDDfqcSkH3MkVyqoA4I8V2FkW7 romain"
  ];
  
  ...
}
~~~
First, we **disable the network predictable interface names** since we don't know yet how they will be named after NixOS takes over Debian.
We want our ethernet access to be named eth0 to be able to specify the useDHCP option for it for now.
That's important if you don't want your machine to lose access to internet after the reboot we'll do.  
Second, we enable openssh daemon to be able to login to our server after NixOS takeover, the current root password won't be kept so we specify our public key for ssh.  
If you want to login as root using a password, you can uncomment the lines, I've given the needed command to setup "nixosdemo" as initial root password.
You'll also need to change the **permitRootLogin** option to **"yes"** as, by default, you won't be able to login as root via ssh since the default option is **"prohibit-password"**. See [here](https://nixos.org/nixos/options#permitrootlogin).  

Next, we'll add a small tweak to our **hardware-configuration.nix** by specifying the device GRUB will use:

~~~nix
# Do not modify this file!  It was generated by ‘nixos-generate-config’
# and may be overwritten by future invocations.  Please make changes
# to /etc/nixos/configuration.nix instead.
{ config, lib, pkgs, ... }:

{
  ...
  
  boot.loader.grub.device = "/dev/sda";
  
  ...
}
~~~
We can now proceed with the remaining steps documented in the manual.  
First, we build the system environment
~~~bash
nix-env -p /nix/var/nix/profiles/system -f '<nixpkgs/nixos>' -I nixos-config=/etc/nixos/configuration.nix -iA system
~~~
We change ownership of the /nix tree to root
~~~bash
sudo chown -R 0.0 /nix
~~~
We create the /etc/NIXOS file which indicates that we are running a NixOS installation.
~~~bash
sudo touch /etc/NIXOS
~~~
We create a NIXOS\_LUSTRATE file which is how we tell NixOS to clean the root partition at boot.
The content of this file indicates which locations the cleaning process must preserve, so we add our **/etc/nixos** directory to it since it contains our **configuration.nix** and **hardware-configuration.nix** files.
~~~bash
sudo touch /etc/NIXOS_LUSTRATE
echo etc/nixos | sudo tee -a /etc/NIXOS_LUSTRATE
~~~
**If your hosting provider has a snapshot option, now would be a good time to backup the state of your machine in case anything goes wrong so that you won't have to get though the whole process again.**  
We now move the Debian boot files and use NixOS **switch-to-configuration** to activate the new system at next boot.
~~~bash
sudo mv -v /boot /boot.bak
sudo /nix/var/nix/profiles/system/bin/switch-to-configuration boot
~~~
Switch back to root and reboot
~~~bash
exit
reboot
~~~

Now, if everything goes well, you should be able to ssh as root to your new NixOS machine
~~~bash
[romain@clevo-N141ZU:~]$ ssh root@159.69.155.132
[root@nixos:~]# uname -a
Linux nixos 5.4.45 #1-NixOS SMP Sun Jun 7 11:18:52 UTC 2020 x86_64 GNU/Linux
~~~

You can now remove the old Debian files placed in **/old-root**
~~~bash
rm -rf /old-root
~~~

Update your machine using NixOS machine
~~~bash
nixos-rebuild switch --upgrade
~~~

Clean the store
~~~bash
nix-collect-garbage -d
~~~

Congratulations ! You now have a clean **NixOS 20.03** installation.
Again, if your hosting provider has a snapshot options, **now would be a good time to make a snapshot so that you could reuse that setup on multiple servers**.

### Alternative ways to get NixOS on your server

The above method has served us well since we were fine with keeping the existing partitionning scheme. 
In case you have more specific needs, you may have to resort to other methods, I'll list some here to get you started:
* [NixOS-infect](https://github.com/elitak/nixos-infect)
* [kexec method](https://nixos.wiki/wiki/Install_NixOS_on_Scaleway_X86_Virtual_Cloud_Server) geared toward Scaleway servers but should be easily tweaked to fit your provider.
* Some providers provide out of the box NixOS installations, that's the case with Hetzner but I wanted to demonstrate a more involved process

## A simple Haskell web service

I've put up a simple haskell web app which you can see on its [github repository](https://github.com/aveltras/deploy-app-with-nixos).
It uses [hedis](https://hackage.haskell.org/package/hedis) to access a Redis server and [hasql](https://hackage.haskell.org/package/hasql) to provide PostgreSQL access.
The database schema is handled by an external migration tool written in Go : [dbmate](https://github.com/amacneil/dbmate).  
The app is simply incrementing two values each time you visit the root path. It's not what you'd call pretty but this isn't the purpose of this post.  
Here's the code from **Main.hs**:

~~~haskell
{-# LANGUAGE LambdaCase          #-}
{-# LANGUAGE OverloadedStrings   #-}
{-# LANGUAGE QuasiQuotes         #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE TemplateHaskell     #-}

module Main where

import qualified Data.ByteString          as B
import qualified Data.ByteString.Char8    as C8
import qualified Data.ByteString.Lazy     as BL
import           Data.Either              (fromRight)
import           Data.Maybe               (fromMaybe)
import qualified Database.Redis           as Redis
import           GHC.Int
import qualified Hasql.Connection         as Hasql
import qualified Hasql.Session            as Hasql
import qualified Hasql.Statement          as Hasql
import           Hasql.TH
import           Network.HTTP.Types
import           Network.Wai
import           Network.Wai.Handler.Warp
import           System.Environment       (getEnv)

main :: IO ()
main = do

  port :: Int <- read <$> getEnv "APP_PORT"
  dbConnStr <- getEnv "DATABASE_URL"

  redisConn <- Redis.checkedConnect Redis.defaultConnectInfo
  sqlConn <- either (error . C8.unpack . fromMaybe "db connection error") id <$> Hasql.acquire (C8.pack dbConnStr)

  run port $ app $ server sqlConn redisConn

app :: (B.ByteString -> IO (Status, BL.ByteString)) -> Application
app handler request sendResponse = do
  (status, bs) <- handler $ rawPathInfo request
  sendResponse $ responseLBS status [(hContentType, "text/plain")] bs

server :: Hasql.Connection -> Redis.Connection -> B.ByteString -> IO (Status, BL.ByteString)
server sqlConn redisConn = \case
  "/"       -> do
    sqlVal <- getSqlValue sqlConn
    redisVal <- getRedisValue redisConn
    pure (ok200, "Current redis value: " <> BL.fromStrict redisVal <> "\nCurrent sql value: " <> (BL.fromStrict . C8.pack . show) sqlVal)
  _         -> pure (notFound404, "not found")

getSqlValue :: Hasql.Connection -> IO Int32
getSqlValue sqlConn = do

  maybeInt <- fmap (either (error . show) id) $ flip Hasql.run sqlConn $
    Hasql.statement () [maybeStatement|
      SELECT intval :: int4
      FROM value
    |]

  case maybeInt of

    Nothing  -> do
      res <- flip Hasql.run sqlConn $ Hasql.statement 0
        [singletonStatement|
          INSERT INTO value (intval)
          VALUES ($1 :: int4)
          RETURNING intval :: int4
        |]

      pure $ unwrap res

    Just int -> do
      res <- flip Hasql.run sqlConn $ Hasql.statement ()
        [singletonStatement|
          UPDATE value SET
          intval = intval + 1 :: int4
          RETURNING intval :: int4
        |]

      pure $ unwrap res


  where
    unwrap :: Either Hasql.QueryError a -> a
    unwrap = \case
      Left err -> error $ show err
      Right a -> a

getRedisValue :: Redis.Connection -> IO B.ByteString
getRedisValue redisConn =
  Redis.runRedis redisConn $ do
    bsVal <- unwrap <$> Redis.get "app:strval"
    case bsVal of
      Nothing -> do
        Redis.set "app:strval" ""
        pure ""
      Just val   -> do
        let newVal = val <> "x"
        _ <- unwrap <$> Redis.set "app:strval" newVal
        pure newVal

  where
    unwrap :: Either Redis.Reply a -> a
    unwrap = \case
      Left err -> error $ show err
      Right val -> val

~~~

As you can see, we'll need configuration values provided as environment variables here, for the port on which to serve our app and for the PostgreSQL connection string.

Let's run our app locally to get a quick sense of what's provided:

~~~bash
cabal run server
~~~

Now in another shell

~~~bash
[romain@clevo-N141ZU:~]$ curl http://localhost:3003
Current redis value: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Current sql value: 67
[romain@clevo-N141ZU:~]$ curl http://localhost:3003
Current redis value: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Current sql value: 68
~~~

Later in this blog post, we'll demonstrate how to package our app to run it on NixOS with its required dependencies.

## Introducing NixOS modules

Now that you have a NixOS installation running, it's time to explain how it works. 
We'll build on the [previous blog post](/en/blog/setting-up-a-haskell-development-environment-with-nix/) so I won't rehash the basics of Nix here, feel free to take a break and read it first if you haven't done so already and are unfamiliar with Nix.  
**NixOS modules** are the building blocks from which you assemble your desired system configuration. Everything in NixOS is a module, from your OS configuration files we encountered in the previous section to the available options at [NixOS options](https://nixos.org/nixos/options.html).  

The basic structure of a module is the following:

~~~nix
{ config, pkgs, ... }:
{
  imports = [
    # paths to other modules
  ];

  options = {
    # option declarations
  };

  config = {
    # option definitions
  };
}
~~~

The first line is optional but it turns out you'll find it in most non trivial NixOS modules. 
This is the way you can access your global machine configuration and other things such as the pkgs of your subscribed channel.
Those are automatically provided by the NixOS configuration building script, you don't have to pass them around yourself.
For example, the **config** parameter here is a reference to **the whole configuration as it's being built**, this enables you to configure your module differently based on the configuration of other modules.  
**Let's know explain what each of the 3 sections is used for:**
* **imports** is a way to include other modules in your configuration. We've already met this with our **configuration.nix** above which is the entry point of our system configuration. There, we imported our **hardware-configuration.nix** which itself imports another NixOS module
~~~nix
# /etc/nixos/configuration.nix
imports =
  [ # Include the results of the hardware scan.
    ./hardware-configuration.nix
  ];
  
# /etc/nixos/hardware-configuration.nix
imports =
  [ <nixpkgs/nixos/modules/profiles/qemu-guest.nix>
  ];
~~~
NixOS is responsible for merging all those configurations and providing them the things they need like **config**, **pkgs**..
* **options** is the way we can provide configuration options for our modules. The first of them often being a boolean **enable** option which will determine in the next section if the module is to be activated or not.
NixOS provides a basic type system we can leverage to ensure the values provided as configuration to our module fullfill some requirements. We'll explain it further with a simple module example below but if you want to learn more, you can refer to the [section about option types](https://nixos.org/nixos/manual/index.html#sec-option-types) in the manual.
* **config** is the way our module provide its functionality to the whole system. It is usually guarded by a boolean test on the **enable** option as we'll see in the example below.

### A quick example : MailHog mail catching facility

Let's deepen our understanding with a small example taken from the available NixOS services. 
I've selected the [MailHog](https://github.com/mailhog/MailHog) preconfigured module.
This is a tool you'd use when NixOS is your development machine since **NixOS is not only geared toward servers, you can use it as your desktop daily driver too**, that's what I personally use.
Alright, let's search for mailhog in the available options, you'll find [two results](https://nixos.org/nixos/options.html#mailhog). If you click on one of them, you'll see that the docs link to the module in the github repo: [https://github.com/NixOS/nixpkgs/blob/release-20.03/nixos/modules/services/mail/mailhog.nix](https://github.com/NixOS/nixpkgs/blob/release-20.03/nixos/modules/services/mail/mailhog.nix).
I'll reproduce the module content here in case it's modified after this article is written.
~~~nix
# https://raw.githubusercontent.com/NixOS/nixpkgs/release-20.03/nixos/modules/services/mail/mailhog.nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.mailhog;
in {
  ###### interface

  options = {

    services.mailhog = {
      enable = mkEnableOption "MailHog";
      user = mkOption {
        type = types.str;
        default = "mailhog";
        description = "User account under which mailhog runs.";
      };
    };
  };


  ###### implementation

  config = mkIf cfg.enable {

    users.users.mailhog = {
      name = cfg.user;
      description = "MailHog service user";
      isSystemUser = true;
    };

    systemd.services.mailhog = {
      description = "MailHog service";
      after = [ "network.target" ];
      wantedBy = [ "multi-user.target" ];
      serviceConfig = {
        Type = "simple";
        ExecStart = "${pkgs.mailhog}/bin/MailHog";
        User = cfg.user;
      };
    };
  };
}
~~~
There we can see that the two options found in the browsable catalog correspond to the two entries in the **options** section of the module.
We have the classic **services.mailhog.enable** option which dictates if the module is in use and we have a **services.mailhog.user** option which we can leverage to specify which NixOS user we want the service to be ran under.
The **user** option is of type **types.str** which means it should be a string and it provides a default value of **mailhog** in case we don't specify one when enabling the module in our global configuration.  
We then have the **config section** guarded by a **mkIf cfg.enable** clause which means the module won't have any effect if it's included but not enabled explicitly in out configuration (or in another module enabled in our config which requires it).
Activating the **MailHog module** will have two effects:
* **First**, it will create a new user on the system
* **Second**, it will create a systemd services to run the MailHog app automatically. It leverages the mailhog package present in Nixpkgs which you can find [here](https://github.com/NixOS/nixpkgs-channels/blob/nixos-20.03/pkgs/servers/mail/mailhog/default.nix).
Right, this will conclude our introduction to NixOS modules, we'll take some inspiration from the MailHog module to create the module for our app.  

You can find more information about NixOS modules here:   
* [NixOS wiki module entry](https://nixos.wiki/wiki/Module)  
* [NixOS manual section about writing modules](https://nixos.org/nixos/manual/index.html#sec-writing-modules)

## Packaging your app as a NixOS module
Right, equipped with your fresh knowledge, now's the time to package our app as a NixOS module.
I won't paste all the existing code here so feel free to browse the sources directly on Github [deploy-app-with-nixos](https://github.com/aveltras/deploy-app-with-nixos)
Our **default.nix** exposes 2 attributes of interest for our current endeavour:
* **server** which is our haskell package
* **migrations** which is a simple package containing our sql migration scripts  
Let's build those locally so that you get a good intuition of what that means.

**First**, the server package..

~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ nix-build -A server
*** Elided build ouput for brievity ***
/nix/store/mk8ismjdd7wd254as18za982wjb5l26c-server-0.1.0
~~~

As usual, the nix-build in its default invocation creates a **result** symlink in the current directory we can inspect.

~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ tree result
result
└── bin
    └── server

1 directory, 1 file
~~~

**Second**, the migrations package..

~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ nix-build -A migrations
unpacking 'https://github.com/NixOS/nixpkgs-channels/archive/nixos-20.03.tar.gz'...
/nix/store/isfy6krvc606p48alcxw6dkk7jx2b7fs-mkMigrations
~~~
~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ tree result
result
└── 20200611140605_setup.sql

0 directories, 1 file
~~~
The resulting package contains only our migrations files.  
We have all we need to start building our module.

Let's create a basic **module.nix** in the source directory of our haskell package.
~~~nix
{ config, lib, pkgs, ... }:

with lib;

let
  
  cfg = config.services.myapp;

in {

  options = {
    services.myapp.enable = mkEnableOption "My App";
  };
  
  config = mkIf cfg.enable {

    services = {
      postgresql = {
        enable = true;
        enableTCPIP = true;
      };

      redis.enable = true;
    };
    
  };
}
~~~
In its current form, our module only defines a **services.myapp.enable** option which, when given **true** in our machine configuration will enable the **redis** and **postgresql** modules provided by NixOS.
As usual, you can browse the options to learn more about the available options for those two modules:
* [Redis](https://nixos.org/nixos/options.html#services.redis.)
* [PostgreSQL](https://nixos.org/nixos/options.html#services.postgresql)

An important thing to notice is the **cfg = config.services.myapp;** binding, that's how the module will reference its own configuration throughout the file. You'll see what I mean in a moment.

If you were to use this module in its current state, enabling it would simply enable a PostgreSQL and a Redis service on your machine.

Let's add our server package to the mix with a first round of modifications.

~~~nix
{ config, lib, pkgs, ... }:

with lib;

let

  pkg = import ./.;

  server = pkg.server;
  
  ...

in {

  ...
  
  config = mkIf cfg.enable {
    
    ...
    
    services = {
      postgresql = {
        enable = true;
        enableTCPIP = true;
      };

      redis.enable = true;
    };
    
    systemd.services = {
      
      myapp = {
        wantedBy = [ "multi-user.target" ];
        description = "Start my app server.";
        after = [ "network.target" ];
        
        serviceConfig = {
          Type = "simple";
          User = "myapp";
          ExecStart = ''${server}/bin/server'';
          Restart = "always";
          KillMode = "process";
        };
      };
    };
    
  };
}
~~~

We now have imported our **default.nix** in the module and bound the server variable to our haskell server package.
We have also added a systemd service for your web server.  
You can find information on defining systemd services by looking at the options [here](https://nixos.org/nixos/options.html#systemd.services.%3Cname%3E).

If you were to use this module right now, the app would crash since it isn't given its needed environment variables and there is currently no database created in PostgreSQL.

Let's make a new round of modifications.

~~~nix
{ config, lib, pkgs, ... }:

with lib;

let

  ...
  
  migrations = pkg.migrations;
  
  ...

in {

  options = {
    services.myapp = {

      ...

      user = mkOption {
        type = types.str;
        default = "myapp";
        description = "User account under which my app runs.";
      };

      port = mkOption {
        type = types.nullOr types.int;
        default = 3003;
        example = 8080;
        description = ''
          The port to bind my app server to.
        '';
      };
    };
  };
  
  config = mkIf cfg.enable {

    users.users.${cfg.user} = {
      name = cfg.user;
      description = "My app service user";
      isSystemUser = true;
    };

    networking.firewall.allowedTCPPorts = [ cfg.port ];

    services = {
      postgresql = {

        enable = true;

        enableTCPIP = true;
        ensureDatabases = [ cfg.user ];

        ensureUsers = [
          {
            name = cfg.user;
            ensurePermissions = {
              "DATABASE ${cfg.user}" = "ALL PRIVILEGES";
            };
          }
        ];

        authentication = pkgs.lib.mkOverride 10 ''
          local sameuser all peer
          host sameuser all ::1/32 trust
        '';
      };

      redis.enable = true;
    };
    
    systemd.services = {

      db-migration = {
        description = "DB migrations script";
        wantedBy = [ "multi-user.target" ];
        after = [ "postgresql.service" ];
        requires = [ "postgresql.service" ];
        
        serviceConfig = {
          User = cfg.user;
          Type = "oneshot";
          ExecStart = "${pkgs.dbmate}/bin/dbmate -d ${migrations} --no-dump-schema up";
        };
      };
      
      myapp = {
        wantedBy = [ "multi-user.target" ];
        description = "Start my app server.";
        after = [ "network.target" ];
        requires = [ "db-migration.service" ];
        
        serviceConfig = {
          Type = "simple";
          User = cfg.user;
          ExecStart = ''${server}/bin/server'';
          Restart = "always";
          KillMode = "process";
        };
      };
    };
  };
}

~~~
There's a lot of changes here. We will now be using our **migrations** packages. 
We have also defined configuration options for our module, **we can now specify the system user and the port to use**.
Those two are referenced in the rest of the file thanks to the **cfg** mentionned earlier. 
We have opened the firewall port in use by our app for it to be accessible from the outer world.
Our PostgreSQL configuration now **ensures that a user and a database with the given username will exist and also configure permissions accordingly**.
It also showcases how to specify a **specific authentication scheme**, here, PostgreSQL will enable both local access (using unix sockets) and access through the local network to the database named like the user trying to access it.
Our previous server systemd service now requires another service before running, the **db-migration service**, which we've just configured. This will ensure that we access an up to date database schema with our app.
The migration service itself requires the PostgreSQL service.  
All this is not functional yet since we still need to provide their configuration values to our services, let's do this with a last round of modifications.

~~~nix
{ config, lib, pkgs, ... }:

with lib;

let
  ...
in {

  ...
  
  config = mkIf cfg.enable {

    ...
    
    systemd.services = {

      db-migration = {

        ...

        environment = {
          DATABASE_URL = "postgres://${cfg.user}@localhost:5432/myapp?sslmode=disable";
        };
        
        ...

      };
      
      myapp = {

        ...

        environment = {
          APP_PORT = toString cfg.port;
          DATABASE_URL = "postgres:///${cfg.user}";
        };
        
        ...
        
      };
    };
  };
}
~~~

We are now providing our two services with their configuration using the environment attribute.
The migration service will connect through the local network as **dbmate** doesn't yet support connecting through a unix socket. Our app will connect through a socket.

Right, this conclude our module definition.  
**A thing to note here is that we didn't have to handle providing configuration files containing secrets.** You can include them directly in your configuration but know that those files will be put in the nix store somewhere and the store is readable by other users on the machine, something to keep in mind if you're using a shared environment.
**Solutions exist for this problem though**, the tool we'll use in the next section provide some functionality to address this problem.

## Deploying with Morph

In order to deploy our configuration, we'll leverage a tool called Morph, you can find more information on its [github repository](https://github.com/DBCDK/morph).
Other solutions exist such as [Nixops](https://github.com/NixOS/nixops) or you could even use hand rolled scripts as described in [Industrial-strength Deployments in Three Commands](https://vaibhavsagar.com/blog/2019/08/22/industrial-strength-deployments/).
I've found Morph simpler to use than Nixops and, more importantly, it doesn't require you to keep deployment state files around.

Morph is available in the nixpkgs channel so let's install it
~~~bash
nix-env -i morph
~~~
At this point, you should copy your server's **/etc/nixos/configuration.nix** and **/etc/nixos/hardware-configuration.nix** locally as deploying will replace them.  
For this example, I've created a **nixos** directory in the project but you may want to store the configurations for your remote servers in their own repositorties, especially if you have multiple app running on a single server. We'll keep things simple here, so let's copy the two configuration files in the **nixos** directory.  

We'll create a **deploy.nix** in the project directory with the following content:

~~~nix
let

  pkgs = import (builtins.fetchTarball {
    url = "https://github.com/NixOS/nixpkgs-channels/archive/nixos-20.03.tar.gz";
  }) {};

in {

  network =  {
    inherit pkgs;
    description = "My NixOS server";
  };

  "159.69.155.132" = { config, pkgs, ... }: {

    deployment = {
      targetUser = "root";
    };

    imports = [
      ./nixos/configuration.nix
      ./module.nix
    ];

    services.myapp = {
      enable = true;
      port = 8080;
    };
    
  };
}
~~~

It's a simple configuration as required by Morph. 
We can deploy to multiple machines but here we'll only need one which we'll address by its IP address.
We specify that deployments should be done with the root user and we simply import our existing **configuration.nix** (which will itself import **hardware-configuration.nix**) and our module.
We enable our service and specify a different port than the default one.
This could have been done in the **configuration.nix** but that's not of great importance here, you can organize your configurations how you want.
We also pin the **nixpkgs version** to the 20.03 archive since that's what is already on our server (subscribed to the 20.03 branch), this will prevent you to upgrade your whole server to a newer NixOS involuntarily (if for example, you're running nixos-unstable on your **desktop machine**).

Let's build our server configuration locally first:
~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ morph build deploy.nix
Selected 1/1 hosts (name filter:-0, limits:-0):
	  0: 159.69.155.132 (secrets: 0, health checks: 0)

building '/nix/store/kb222zkxdfj9bklayr5q8ma141jjwznx-cabal2nix-server.drv'...
*** Elided output for brievity ***
nix result path:
/nix/store/vxg1plm65wzvxk6vz4k1n1ks9gy5jgpq-morph
~~~
We can see what the system build looks like by browsing the store entry the build returned.

~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ tree /nix/store/vxg1plm65wzvxk6vz4k1n1ks9gy5jgpq-morph/159.69.155.132
/nix/store/vxg1plm65wzvxk6vz4k1n1ks9gy5jgpq-morph/159.69.155.132
├── activate
├── append-initrd-secrets -> /nix/store/z6kk89lds89jawpv0zy471gfq56hf43j-append-initrd-secrets/bin/append-initrd-secrets
├── bin
│   └── switch-to-configuration
├── configuration-name
├── etc -> /nix/store/ls4qbbl0h6qkd6bn71n798x2i8sys71f-etc/etc
├── extra-dependencies
├── fine-tune
├── firmware -> /nix/store/933miznriml41c98rrid6zxpyqqjgqx2-firmware/lib/firmware
├── init
├── init-interface-version
├── initrd -> /nix/store/46x17f67k9rmhqpmrysv08bpg1ndp5gg-initrd-linux-5.4.45/initrd
├── kernel -> /nix/store/2rwn0zkprhcjr9psgrs79ci2jgs6i2f6-linux-5.4.45/bzImage
├── kernel-modules -> /nix/store/m6v1a0bk5x70a63nnqbfiqqs7p7mx9ys-kernel-modules
├── kernel-params
├── nixos-version
├── sw -> /nix/store/h2h3jncpysal5q05krnf7pk6fbp0dnvv-system-path
├── system
└── systemd -> /nix/store/vfzp1mavwiz5w3v10hs69962k0gwl26c-systemd-243.7

7 directories, 12 files
~~~

It's time to deploy our configuration. For this we'll run the following:
~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ morph deploy deploy.nix test
Selected 1/1 hosts (name filter:-0, limits:-0):
	  0: 159.69.155.132 (secrets: 0, health checks: 0)

/nix/store/vxg1plm65wzvxk6vz4k1n1ks9gy5jgpq-morph
nix result path:
/nix/store/vxg1plm65wzvxk6vz4k1n1ks9gy5jgpq-morph

Pushing paths to 159.69.155.132 (root@159.69.155.132):
	* /nix/store/a8q5qb4pqfnbdxbp0hy4wws82gfd02v6-nixos-system-159.69.155.132-20.03post-git
Enter passphrase for key '/home/romain/.ssh/id_ed25519':
[52 copied (69.7 MiB)]

Executing 'test' on matched hosts:

** 159.69.155.132
Enter passphrase for key '/home/romain/.ssh/id_ed25519':
stopping the following units: nscd.service, systemd-sysctl.service
NOT restarting the following changed units: systemd-fsck@dev-disk-by\x2duuid-3143\x2d1D20.service
activating the configuration...
setting up /etc...
reloading user units for root...
setting up tmpfiles
reloading the following units: dbus.service, firewall.service
starting the following units: nscd.service, systemd-sysctl.service
the following new units were started: myapp.service, postgresql.service, redis.service

Running healthchecks on 159.69.155.132 (159.69.155.132):
Health checks OK
Done: 159.69.155.132
~~~

Here I've deployed using **test**, you have several options for the deploy command as stated on Morph github readme.  
_Notably, morph deploy requires a <switch-action>. The switch-action must be one of dry-activate, test, switch or boot corresponding to nixos-rebuild arguments of the same name. Refer to the [NixOS manual](https://nixos.org/nixos/manual/index.html#sec-changing-config) for a detailed description of switch-actions._

We can now test it from out desktop machine:
~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ curl http://159.69.155.132:8080/
Current redis value: x
Current sql value: 1
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ curl http://159.69.155.132:8080/
Current redis value: xx
Current sql value: 2
~~~

**I should probably state here that it didn't work right directly. For some reason, the app server could not connect to the PostgreSQL socket.**
That was somehow easy to debug though, I've just ssh'ed on the server and ran :
~~~bash
[root@159:~]# systemctl status myapp.service
● myapp.service - Start my app server.
   Loaded: loaded (/nix/store/74pg5qwlikn130b1fgijkwp9bqvccdn1-unit-myapp.service/myapp.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-06-12 17:21:55 CEST; 3min 30s ago
 Main PID: 2061 (server)
       IP: 1.1K in, 1.3K out
    Tasks: 1 (limit: 2316)
   Memory: 1.5M
      CPU: 528ms
   CGroup: /system.slice/myapp.service
           └─2061 /nix/store/b6va2qkkf8pdqc59ndhvxcrmdc8rbm58-server-0.1.0/bin/server

juin 12 17:24:29 159.69.155.132 server[2061]: could not connect to server: No such file or directory
juin 12 17:24:29 159.69.155.132 server[2061]:         Is the server running locally and accepting
juin 12 17:24:29 159.69.155.132 server[2061]:         connections on Unix domain socket "/run/postgresql/.s.PGSQL.5432"?
juin 12 17:24:29 159.69.155.132 server[2061]: CallStack (from HasCallStack):
juin 12 17:24:29 159.69.155.132 server[2061]:   error, called at src/Main.hs:32:22 in main:Main
juin 12 17:25:14 159.69.155.132 server[2061]: could not connect to server: No such file or directory
juin 12 17:25:14 159.69.155.132 server[2061]:         Is the server running locally and accepting
juin 12 17:25:14 159.69.155.132 server[2061]:         connections on Unix domain socket "/run/postgresql/.s.PGSQL.5432"?
juin 12 17:25:14 159.69.155.132 server[2061]: CallStack (from HasCallStack):
juin 12 17:25:14 159.69.155.132 server[2061]:   error, called at src/Main.hs:32:22 in main:Main
~~~

Restarting PostgreSQL service and then the myapp.service fixed it.

**To finalize your configuration, don't forget to morph deploy with the switch option instead of test.**

As mentioned previously, Morph provides some secrets handling functionality which you can learn about on their [github readme](https://github.com/DBCDK/morph#secrets).
That's good to know if you need it, which is bound to happen if you use NixOS for any non trivial application.

**Alright that's the bulk of what you need to know to be able to deploy your applications to a NixOS server.
We'll explore in the last section how you can put Nginx in front of it and how to setup ssl with Let's Encrypt.**

## Bonus: Configuring Nginx and Let's encrypt

NixOS provides out of the box support for generating Let's Encrypt certificates so we'll leverage this. We just need to tweak our configuration a bit.

~~~nix
let

  pkgs = import (builtins.fetchTarball {
    url = "https://github.com/NixOS/nixpkgs-channels/archive/nixos-20.03.tar.gz";
  }) {};

in {

  network =  {
    inherit pkgs;
    description = "My NixOS server";
  };

  "159.69.155.132" = { config, pkgs, ... }: {

    ...

    networking = {
      domain = "demo.romainviallard.dev";
      firewall.allowedTCPPorts = [ 80 443 ];
    };
    
    security.acme = {
      acceptTerms = true;
      # validMinDays = 999;
      # server = "https://acme-staging-v02.api.letsencrypt.org/directory"; # uncomment this to use the staging server
      email = "romain@romainviallard.dev";
      certs."demo.romainviallard.dev".extraDomains = {
        # "demo2.romainviallard.dev" = null;
      };
    };
    
    ...

    services.nginx = {
      enable = true;

      recommendedGzipSettings = true;
      recommendedOptimisation = true;
      recommendedProxySettings = true;
      recommendedTlsSettings = true;

      # https://nixos.org/nixos/manual/#module-security-acme-nginx
      virtualHosts = {
        
        "demo.romainviallard.dev" = {
          enableACME = true;
          forceSSL = true;
          locations."/" = {
            proxyPass = "http://127.0.0.1:${toString config.services.myapp.port}";
          };
        };

        # "demo2.romainviallard.dev" = {
        #   useACMEHost = "demo.romainviallard.dev";
        #   forceSSL = true;
        #   locations."/" = {
        #     proxyPass = "http://127.0.0.1:${toString config.services.myapp.port}";
        #   };
        # };
      };
    };
    
  };
}
~~~
We've opened the 80 and 443 ports on our firewall in order to let web users access our app.  
As you can see, we directly reference the configuration value **config.services.myapp.port** here even if we are out of our module definition file.
Here, we leverage Let's Encrypt to generate an ssl certificate for the **demo.romainviallard.dev** domain.
To generate a single certificate for multiple domains, you assign **extraDomains** to your main domain, I didn't do it here since I haven't configured a dns entry for the **demo2.romainviallard.dev** subdomain and this prevents the Let's encrypt process from succeeding.
We then enable the **Nginx** service using its recommended settings and we add a virtual host which forwards all requests to our internal app server.
The important attribute here is **enableACME** which is what you must use with **true** on your main domain (the one the certificate will primarily generated for). In case you have several domains using the same certificate, you must use the **useACMEHost** instead of **enableACME** attribute on secondary domains, giving it the primary domain.
The **security.acme.validMinDays** attribute can be uncommented when you need to force a refresh of the certificate, in case you've added a domain to it for example.
The **security.acme.server** attribute can be used to provide a different letsencrypt server, when not provided, it uses the production one. You should use a staging server when making configuration tests as to not be throttled by Let's Encrypt because of too many requests.

Let's check this still works.
~~~bash
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ curl https://demo.romainviallard.dev/
Current redis value: xxxxxxxxxxx
Current sql value: 11
[romain@clevo-N141ZU:~/Code/deploy-app-with-nixos]$ curl https://demo.romainviallard.dev/
Current redis value: xxxxxxxxxxxx
Current sql value: 12
~~~

All is fine !

Alright, that's it for today, I hope this blog post has given you a nice overview of the niceties the Nix ecosystem brings to the table. As usual, if you have any questions, feel free open an issue.
