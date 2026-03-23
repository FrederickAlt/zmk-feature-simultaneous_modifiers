if(CONFIG_ZMK_KEY_LAYER_DEFER)
    zephyr_library()
    zephyr_library_sources(src/key_layer_defer.c)
endif() 
