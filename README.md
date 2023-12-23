# PID_SYS_CALL
Adding A System Call To The Linux Kernel (5.15.1) In Ubuntu (20.04 LTS) 

## Section 1 - Preparation 
# 1.1 - Fully update your operating system
~$ sudo apt update && sudo apt upgrade -y

# 1.2 - Download and install the essential packages to compile kernels.
~$ sudo apt install build-essential libncurses-dev libssl-dev libelf-dev bison flex -y

** If would rather use vim or any other text editor instead of nano, below is an example of how you install it.
~$ sudo apt install vim -y

# 1.3 - Clean up your installed packages.
~$ sudo apt clean && sudo apt autoremove -y

# 1.4 - Download the source code of the latest stable version of the Linux kernel (which is 5.15.1 as my chosen) to your home folder.
~$ wget -P ~/  https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.1.tar.xz

# 1.5 - Unpack the tarball you just downloaded to your home folder.
~$ tar -xvf ~/linux-5.15.1.tar.xz -C ~/


## Section 2 - Creation 
# 2.1 - Check the version of your current kernel & reboot ur VM System.
~$ uname -r
5.15.0-86-generic

& reboot ur VM System.

# 2.2 - Change your working directory to the root directory of the recently unpacked source code.
~$ cd ~/linux-5.8.1/

# 2.3 - Create the home directory of your system call.

Decide a name for your system call, and keep it consistent from this point onwards. I have chosen "systemcall_custom".
~/linux-5.15.1$ mkdir systemcall_custom

# 2.4 - Create a C file for your system call.

Create the C file with the following command.
~/linux-5.15.1$ nano systemcall_custom/systemcall_custom.c


#include <linux/kernel.h>
#include <linux/syscalls.h>
//sh
#include <linux/file.h>
#include <linux/fs.h>
#include <linux/slab.h>
#include <asm/uaccess.h>

asmlinkage long __x64_sys_systemcall_custom(pid_t pid, const char __user *filename) {
    mm_segment_t oldfs;
    struct file *file;
    char *buffer;
    loff_t offset = 0;
    int found = 0;

    // Check if PID exists in the file
    oldfs = get_fs();
    set_fs(get_ds());

    file = filp_open(filename, O_RDWR | O_CREAT, 0644);
    if (IS_ERR(file)) {
        set_fs(oldfs);
        return PTR_ERR(file);
    }

    buffer = kmalloc(PAGE_SIZE, GFP_KERNEL);
    if (!buffer) {
        filp_close(file, NULL);
        set_fs(oldfs);
        return -ENOMEM;
    }

    // Read file content and check for existing PID
    while (vfs_read(file, buffer, PAGE_SIZE, &offset) > 0) {
        char *pid_str = strstr(buffer, "PID:");
        if (pid_str && (pid_t) simple_strtol(pid_str + 4, NULL, 10) == pid) {
            found = 1;
            break;
        }
    }

    // If PID not found, add it to the file
    if (!found) {
        offset = 0;
        vfs_write(file, "PID:", 4, &offset);
        vfs_write(file, " ", 1, &offset);
        vfs_write(file, simple_itoa(pid), strlen(simple_itoa(pid)), &offset);
        vfs_write(file, "\n", 1, &offset);
    }

    kfree(buffer);
    filp_close(file, NULL);
    set_fs(oldfs);

    // Print the result
    if (found) {
        pr_info("1\n");
        return 1;
    } else {
        pr_info("0\n");
        return 0;
    }
}

**Ctrl + X to Save it and exit the text editor.







