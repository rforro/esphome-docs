Lambda Magic
============

.. seo::
    :description: Recipes for various interesting things you can do with Lambdas in ESPHome
    :image: language-cpp.svg

Here are a couple recipes for various interesting things you can do with :ref:`Lambdas <config-lambda>` in ESPHome.
These don't require external components and demonstrate how powerful :ref:`Lambdas <config-lambda>` can be.

.. _lambda_magic_pages:

Display pages alternative
-------------------------

Some displays like :ref:`lcd-pcf8574` don't support pages natively, but you can easily implement them
using Lambdas:

.. code-block:: yaml

    display:
      - platform: lcd_pcf8574
        dimensions: 20x4
        address: 0x27
        id: lcd
        lambda: |-
              switch (id(page)){
                case 1:
                  it.print(0, 1, "Page1");
                  break;
                case 2:
                  it.print(0, 1, "Page2");
                  break;
                case 3:
                  it.print(0, 1, "Page3");
                  break;
              }

    globals:
    - id: page
      type: int
      initial_value: "1"

    interval:
    - interval: 5s
      then:
        - lambda: |-
            id(page) = (id(page) + 1);
            if (id(page) > 3) {
              id(page) = 1;
            }


.. _lambda_magic_udp_sender:

Send UDP commands
-----------------

There are various network devices which can be commanded with UDP packets containing command strings.
You can send such UDP commands from ESPHome using a Lambda in a script.

.. code-block:: yaml

    script:
    - id: send_udp
      parameters:
        msg: string
        host: string
        port: int
      then:
        - lambda: |-
              int sock = ::socket(AF_INET, SOCK_DGRAM, 0);
              struct sockaddr_in destination, source;

              destination.sin_family = AF_INET;
              destination.sin_port = htons(port);
              destination.sin_addr.s_addr = inet_addr(host.c_str());

              // you can remove the next 4 lines if you don't want to set the source port for outgoing packets
              source.sin_family = AF_INET;
              source.sin_addr.s_addr = htonl(INADDR_ANY);
              source.sin_port = htons(64998);  // the source port number
              bind(sock, (struct sockaddr*)&source, sizeof(source));

              int n_bytes = ::sendto(sock, msg.c_str(), msg.length(), 0, reinterpret_cast<sockaddr*>(&destination), sizeof(destination));
              ESP_LOGD("lambda", "Sent %s to %s:%d in %d bytes", msg.c_str(), host.c_str(), port, n_bytes);
              ::close(sock);

    button:
    - platform: template
      id: button_udp_sender
      name: "Send UDP Command"
      on_press:
        - script.execute:
            id: send_udp
            msg: "Hello World!"
            host: "192.168.1.10"
            port: 5000

Tested on both `arduino` and `esp-idf` platforms.

.. _lambda_magic_rf_queues:

Delaying Remote Transmissions
-----------------------------

The solution below handles the problem of RF frames being sent out by :doc:`/components/rf_bridge` (or
:doc:`/components/remote_transmitter`) too quickly one after another when operating radio controlled
covers. The cover motors seem to need at least 600-700ms of silence between the individual code transmissions
to be able to recognize them.

This can be solved by building up a queue of raw RF codes and sending them out one after the other with
(a configurable) delay between them. Delay is only added to the next commands coming from a list of
covers which have to be operated at once from Home Assistant. This is transparent to the system, which
will still look like they operate simultaneously.

.. code-block:: yaml

    rf_bridge:

    number:
    - platform: template
      name: Delay commands
      icon: mdi:clock-fast
      entity_category: config
      optimistic: true
      restore_value: true
      initial_value: 750
      unit_of_measurement: "ms"
      id: queue_delay
      min_value: 10
      max_value: 1000
      step: 50
      mode: box

    globals:
    - id: rf_code_queue
      type: 'std::vector<std::string>'

    script:
    - id: rf_transmitter_queue
      mode: single
      then:
        while:
          condition:
            lambda: 'return !id(rf_code_queue).empty();'
          then:
             - rf_bridge.send_raw:
                 raw: !lambda |-
                   std::string rf_code = id(rf_code_queue).front();
                   id(rf_code_queue).erase(id(rf_code_queue).begin());
                   return rf_code;
             - delay: !lambda 'return id(queue_delay).state;'

    cover:
        # have multiple covers
      - platform: time_based
        name: 'My Room 1'
        disabled_by_default: false
        device_class: shutter
        assumed_state: true
        has_built_in_endstop: true

        close_action:
          - lambda: id(rf_code_queue).push_back("AAB0XXXXX..the.closing.code..XXXXXXXXXX");
          - script.execute: rf_transmitter_queue
        close_duration: 26s

        stop_action:
          - lambda: id(rf_code_queue).push_back("AAB0YXXXX..the.stopping.code..XXXXXXXXXX");
          - script.execute: rf_transmitter_queue

        open_action:
          - lambda: id(rf_code_queue).push_back("AAB0ZXXXX..the.opening.code..XXXXXXXXXX");
          - script.execute: rf_transmitter_queue
        open_duration: 27s

.. _lambda_magic_1button_coover:

One Button Cover Control
------------------------

The configuration below shows how with a single button you can control the motion of a motorized cover
by cycling between: open->stop->close->stop->...

In this example a :doc:`/components/cover/time_based` is used with the GPIO configuration of a Sonoff Dual R2.

.. note::

    Controlling the cover to quickly (sending new open/close commands within a minute of previous commands)
    might cause unexpected behaviour (eg: cover stopping halfway). This is because the delayed relay off
    feature is implemented using asynchronous automations. So every time an open/close command is sent a
    delayed relay off command is added and old ones are not removed.

.. code-block:: yaml

    esp8266:
      board: esp01_1m

    binary_sensor:
    - platform: gpio
      pin:
        number: GPIO10
        inverted: true
      id: button
      on_press:
        then:
          # logic for cycling through movements: open->stop->close->stop->...
          - lambda: |
              if (id(my_cover).current_operation == COVER_OPERATION_IDLE) {
                // Cover is idle, check current state and either open or close cover.
                if (id(my_cover).is_fully_closed()) {
                  id(my_cover).open();
                } else {
                  id(my_cover).close();
                }
              } else {
                // Cover is opening/closing. Stop it.
                id(my_cover).stop();
              }

    switch:
    - platform: gpio
      pin: GPIO12
      interlock: &interlock [open_cover, close_cover]
      id: open_cover
    - platform: gpio
      pin: GPIO5
      interlock: *interlock
      id: close_cover

    cover:
    - platform: time_based
      name: "Cover"
      id: my_cover
      open_action:
        - switch.turn_on: open_cover
      open_duration: 60s
      close_action:
        - switch.turn_on: close_cover
      close_duration: 60s
      stop_action:
        - switch.turn_off: open_cover
        - switch.turn_off: close_cover

Update numeric values from text input
-------------------------------------

Sometimes it may be more confortable to use a :doc:`/components/text/template` to change some numeric values from the user interface.
ESPHome has some nice `helper functions <https://github.com/esphome/esphome/blob/dev/esphome/core/helpers.h>`__ among which
theres's one to convert text to numbers.

In the example below we have a text input and a template sensor which can be updated from the text input field. What the lambda
does, is to parse and convert the text string to a number - which only succeedes if the entered string contains characters
represesenting a float number (such as digits, ``-`` and ``.``). If the entered string contains any other characters, the lambda
will return ``NaN``, which corresponds to ``unknown`` sensor state.

.. code-block:: yaml

    text:
      - platform: template
        name: "Number type in"
        optimistic: true
        min_length: 0
        max_length: 16
        mode: text
        on_value:
          then:
            - sensor.template.publish:
                id: num_from_text
                state: !lambda |-
                  auto n = parse_number<float>(x);
                  return n.has_value() ? n.value() : NAN;

    sensor:
      - platform: template
        id: num_from_text
        name: "Number from text"


Factory reset after 5 quick reboots
-----------------------------------

One may want to restore factory settings (like Wi-Fi credentials set at runtime, or clear restore states) without having to
disassemble or dismount the devices from their deployed location, whilst there's no network access either. The example below
shows how to achieve that using lambdas in a script by triggering the factory reset switch after the system rebooted 5 times
with 10-second timeframes.

.. code-block:: yaml

    # Example config.yaml
    esphome:
      name: "esphome_ld2410"
      on_boot:
        priority: 600.0
        then:
          - script.execute: fast_boot_factory_reset_script
    esp32:
      board: esp32-c3-devkitm-1

    substitutions:
      factory_reset_boot_count_trigger: 5 

    globals:
      - id: fast_boot
        type: int
        restore_value: yes
        initial_value: '0'
    
    script:
      - id: fast_boot_factory_reset_script
        then:
          - if:
              condition:
                lambda: return ( id(fast_boot) >= ${factory_reset_boot_count_trigger});
              then:
                - lambda: |-
                    ESP_LOGD("Fast Boot Factory Reset", "Performing factotry reset");
                    id(fast_boot) = 0;
                    fast_boot->loop();
                    global_preferences->sync();
                - button.press: factory_reset_button
          - lambda: |-
              if(id(fast_boot) > 0)
                ESP_LOGD("Fast Boot Factory Reset", "Quick reboot %d/%d, do it %d more times to factory reset", id(fast_boot), ${factory_reset_boot_count_trigger}, ${factory_reset_boot_count_trigger} - id(fast_boot));
              id(fast_boot) += 1;
              fast_boot->loop();
              global_preferences->sync();
          - delay: 10s
          - lambda: |-
              id(fast_boot) = 0;
              fast_boot->loop();
              global_preferences->sync();
    
    wifi:
      id: wifi_component
      ap:
        ap_timeout: 0s
      reboot_timeout: 0s
    
    captive_portal:
    
    button:
      - platform: factory_reset
        id: factory_reset_button
        name: "ESPHome: Factory reset"


See Also
--------

- :ref:`config-lambda`
- :ref:`automation`

- :ghedit:`Edit`
