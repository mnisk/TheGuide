
#### console

> location = src/system/boot/platform/efi/console.cpp

```

console_init(void)
{
#if 1
	gConsoleNode = &sConsole;

	update_screen_size();
	console_hide_cursor();
	console_clear_screen();
#else
	// FIXME: This does not work because we cannot initialize video before VFS, as it
	// needs to read the driver settings before setting a mode; and also because the
	// heap does not yet exist.
	platform_init_video();
	platform_switch_to_logo();
	gConsoleNode = video_text_console_init(gKernelArgs.frame_buffer.physical_buffer.start);
#endif

	// enable stdio functionality
	stdin = (FILE *)gConsoleNode;
	stdout = stderr = (FILE *)gConsoleNode;

	return B_OK;
}

```
* #if 1 is executed always because, 1 means true so the expressin becomes true.
* [update_screen_size()](/boot/efi/console.md#update_screen_size)
* [console_hide_cursor()](/boot/arch/generic/text_console.md#console_hide_cursor)
* [console_clear_screen()](/boot/arch/generic/text_console.md#console_clear_screen)


## console_wait_for_key

```
int
console_wait_for_key(void)
{
	size_t index;
	efi_status status;
	efi_input_key key;
	efi_event event = kSystemTable->ConIn->WaitForKey;

	do {
		kBootServices->WaitForEvent(1, &event, &index);
		status = kSystemTable->ConIn->ReadKeyStroke(kSystemTable->ConIn, &key);
	} while (status == EFI_NOT_READY);

	if (key.UnicodeChar > 0)
		return (int) key.UnicodeChar;

	switch (key.ScanCode) {
		case SCAN_ESC:
			return TEXT_CONSOLE_KEY_ESCAPE;
		case SCAN_UP:
			return TEXT_CONSOLE_KEY_UP;
		case SCAN_DOWN:
			return TEXT_CONSOLE_KEY_DOWN;
		case SCAN_LEFT:
			return TEXT_CONSOLE_KEY_LEFT;
		case SCAN_RIGHT:
			return TEXT_CONSOLE_KEY_RIGHT;
		case SCAN_PAGE_UP:
			return TEXT_CONSOLE_KEY_PAGE_UP;
		case SCAN_PAGE_DOWN:
			return TEXT_CONSOLE_KEY_PAGE_DOWN;
		case SCAN_HOME:
			return TEXT_CONSOLE_KEY_HOME;
		case SCAN_END:
			return TEXT_CONSOLE_KEY_END;
	}
	return 0;
}
```

## console_check_boot_keys()
```

uint32
console_check_boot_keys(void)
{
	efi_input_key key;

	for (int i = 0; i < 3; i++) {
		// give the user a chance to press a key
		kBootServices->Stall(100000);

		efi_status status = kSystemTable->ConIn->ReadKeyStroke(
			kSystemTable->ConIn, &key);

		if (status != EFI_SUCCESS)
			continue;

		if (key.UnicodeChar == 0 && key.ScanCode == SCAN_ESC)
			return BOOT_OPTION_DEBUG_OUTPUT;
		if (key.UnicodeChar == ' ')
			return BOOT_OPTION_MENU;
	}
	return 0;
}

```

## update_screen_size()

```
static void update_screen_size(void)
{
	size_t width, height;
	size_t area = 0;
	efi_simple_text_output_protocol *ConOut = kSystemTable->ConOut;

	for (int mode = 0; mode < ConOut->Mode->MaxMode; ++mode) {
		if (ConOut->QueryMode(ConOut, mode, &width, &height) == EFI_SUCCESS) {
			if (width * height > area) {
				sConsole.fScreenWidth = width;
				sConsole.fScreenHeight = height;
				sScreenMode = mode;
			}
		}
	}

	ConOut->SetMode(ConOut, sScreenMode);
}
```
