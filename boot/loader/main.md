## loader

## main()

> arg1 = [stage2_args](/boot/structure/stage2_args.md) *args

```

extern "C" int
main(stage2_args *args)
{
	TRACE(("boot(): enter\n"));

	if (heap_init(args) < B_OK)
		panic("Could not initialize heap!\n");

	TRACE(("boot(): heap initialized...\n"));

	// set debug syslog default
#if KDEBUG_ENABLE_DEBUG_SYSLOG
	gKernelArgs.keep_debug_output_buffer = true;
	gKernelArgs.previous_debug_size = true;
		// used as a boolean indicator until initialized for the kernel
#endif

	add_stage2_driver_settings(args);

	platform_init_video();

	// the main platform dependent initialisation
	// has already taken place at this point.

	if (vfs_init(args) < B_OK)
		panic("Could not initialize VFS!\n");

	dprintf("Welcome to the Haiku boot loader!\n");
	dprintf("Haiku revision: %s\n", get_haiku_revision());

	bool mountedAllVolumes = false;

	BootVolume bootVolume;
	PathBlocklist pathBlocklist;

	if (get_boot_file_system(args, bootVolume) != B_OK
		|| (platform_boot_options() & BOOT_OPTION_MENU) != 0) {
		if (!bootVolume.IsValid())
			puts("\tno boot path found, scan for all partitions...\n");

		if (mount_file_systems(args) < B_OK) {
			// That's unfortunate, but we still give the user the possibility
			// to insert a CD-ROM or just rescan the available devices
			puts("Could not locate any supported boot devices!\n");
		}

		// ToDo: check if there is only one bootable volume!

		mountedAllVolumes = true;

		if (user_menu(bootVolume, pathBlocklist) < B_OK) {
			// user requested to quit the loader
			goto out;
		}
	}

	if (bootVolume.IsValid()) {
		// we got a volume to boot from!

		// TODO: fix for riscv64
#ifndef __riscv
		load_driver_settings(args, bootVolume.RootDirectory());
#endif
		status_t status;
		while ((status = load_kernel(args, bootVolume)) < B_OK) {
			// loading the kernel failed, so let the user choose another
			// volume to boot from until it works
			bootVolume.Unset();

			if (!mountedAllVolumes) {
				// mount all other file systems, if not already happened
				if (mount_file_systems(args) < B_OK)
					panic("Could not locate any supported boot devices!\n");

				mountedAllVolumes = true;
			}

			if (user_menu(bootVolume, pathBlocklist) != B_OK
				|| !bootVolume.IsValid()) {
				// user requested to quit the loader
				goto out;
			}
		}

		// if everything is okay, continue booting; the kernel
		// is already loaded at this point and we definitely
		// know our boot volume, too
		if (status == B_OK) {
			if (bootVolume.IsPackaged()) {
				packagefs_apply_path_blocklist(bootVolume.SystemDirectory(),
					pathBlocklist);
			}

			register_boot_file_system(bootVolume);

			if ((platform_boot_options() & BOOT_OPTION_DEBUG_OUTPUT) == 0)
				platform_switch_to_logo();

			load_modules(args, bootVolume);

			gKernelArgs.ucode_data = NULL;
			gKernelArgs.ucode_data_size = 0;
			platform_load_ucode(bootVolume);

			// TODO: fix for riscv64
#ifndef __riscv
			// apply boot settings
			apply_boot_settings();
#endif

			// set up kernel args version info
			gKernelArgs.kernel_args_size = sizeof(kernel_args);
			gKernelArgs.version = CURRENT_KERNEL_ARGS_VERSION;
			if (gKernelArgs.ucode_data == NULL)
				gKernelArgs.kernel_args_size = kernel_args_size_v1;

			// clone the boot_volume KMessage into kernel accessible memory
			// note, that we need to 8-byte align the buffer and thus allocate
			// 7 more bytes
			void* buffer = kernel_args_malloc(gBootVolume.ContentSize() + 7);
			if (!buffer) {
				panic("Could not allocate memory for the boot volume kernel "
					"arguments");
			}

			buffer = (void*)(((addr_t)buffer + 7) & ~(addr_t)0x7);
			memcpy(buffer, gBootVolume.Buffer(), gBootVolume.ContentSize());
			gKernelArgs.boot_volume = buffer;
			gKernelArgs.boot_volume_size = gBootVolume.ContentSize();

			platform_cleanup_devices();
			// TODO: cleanup, heap_release() etc.
			heap_print_statistics();
			platform_start_kernel();
		}
	}

out:
	heap_release(args);
	return 0;
}
Footer

```

* [heap_init()](/boot/loader/heap.md#heap_init)
* [add_stage2_driver_settings(args)](/boot/loader/load_driver_settings.md#add_stage2_driver_settings)
* [platform_init_video()](/boot/efi/video.md#platform_init_video)
* [vfs_init()](/boot/loader/vfs.md#vfs_init)
* bool mountedAllVolumes = false;
* [BootVolume](/boot/loader/vfs.md#BootVolume) bootVolume; // BootVolume object.
* [PathBlocklist](/boot/loader/PathBlocklist.md#PathBlocklist) pathBlocklist;	// PathBlocklist object.
* [get_boot_file_system(args, bootVolume)](/boot/loader/vfs.md#get_boot_file_system) || [platform_boot_options()](/boot/efi/start.md#platform_boot_options) == BOOT_OPTION_MENU

   * [mount_file_systems(args)](/boot/loader/vfs.md#mount_file_systems)
   * [user_menu(bootVolume, pathBlocklist)](/boot/loader/menu.md#user_menu)

* if (bootVolume.IsValid()) {
	* // we got a volume to boot from!

	* // TODO: fix for riscv64
	* #ifndef __riscv
	* [load_driver_settings(args, bootVolume.RootDirectory())](/boot/loader/load_driver_settings.md#load_driver_settings)
	* #endif

	* status_t status;
	* while ((status = [load_kernel](/boot/loader/loader.md#load_kernel)(args, bootVolume)) < B_OK) {



