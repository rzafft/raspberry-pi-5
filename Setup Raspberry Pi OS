# Raspberry Pi OS

1. Write the Raspberry Pi OS to the NVMe SSD (using the Raspberry Pi Imager). Below are the raspbery pi imager configuraiton settings that we seelcted...

  •	Operating System: RASPBERRY PI OS (64 bit)
  •	Storage: ACASIS EC-6606 Media 512 GB
  •	Hostname: rp5-pios
  •	Username: rza
  •	Enable SSH: True (Use password auth: True, Allow public key auth only: False)

2. Boot into the pi os

# About the PiOS filesysten

Root (/)
	•	/bin (essential binaries - commands like ls, cat, etc.)
	•	/sbin (system binaries - admin commands like fdisk, mount, etc.)
	•	/boot (boot files and kernel images - important for pi to start)
	•	/dev (device files - disks, usb, gpio, etc.)
	•	/etc (configurations for system and applications)
	•	/home (user home directories - /home/rza)
	•	/lib (shared libraries needed by binaries)
	•	/media (mount point for removable media - usb sticks, external drives, etc.)
	•	/mnt (temporary mount point - usually used for manual mounting)
	•	/opt (optional software packages installed manually)
	•	/proc (virtual filesystem with process and kernel info)
	•	/run (runtime data like pid files and sockets)
	•	/root (home directory for root user_
	•	/sys (virtual filesystem for kernel objects)
	•	/tmp (temporary iles, cleared on reboot)
	•	/usr (secondary hierarchy for binaries, libraries, documentation, etc)
	•	/usr/lib (libraries used by system programs)
	•	/usr/bin (system wide executables - e.g. python3, Java, etc. never put files here manually, use package manager (apt) instead)
	•	/var (variable data - logs, databases, caches, spool files)
	•	/var/log (system logs, useful for debugging)
	•	/lost+found (files recovered after filesystem corruption)

System-Wide Installations (/usr/bin)
	•	Used by Raspberry Pi OS itself and by other system programs.
	•	These installations require administrative privileges (sudo).
	•	Use system-wide installs (apt) if you want all users to share the same Python installation and packages.

User Installations (/home/rza/.local/bin/)
	•	Used only by the rza user.
	•	Installs don’t require admin privileges.
	•	Safer for experimentation or testing new libraries.
	•	Keeps /usr/bin and /usr/lib untouched.
	•	Use user-level (--user) or virtual environments if you want isolation, safety, and flexibility for your own projects.
    
