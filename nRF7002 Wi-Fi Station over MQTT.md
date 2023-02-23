# nRF7002 Wi-Fi Station over MQTT

## nRF Connect SDK v2.2.0


```main.c
/*
 * Copyright (c) 2022 Nordic Semiconductor ASA
 *
 * SPDX-License-Identifier: LicenseRef-Nordic-5-Clause
 */

/** @file
 * @brief WiFi station sample
 */

#include <zephyr/logging/log.h>
#include <zephyr/types.h>
#include <stddef.h>
#include <string.h>
#include <errno.h>

#include <nrfx_clock.h>
#include <zephyr/kernel.h>
#include <stdio.h>
#include <stdlib.h>
#include <zephyr/shell/shell.h>
#include <zephyr/sys/printk.h>
#include <zephyr/init.h>

#include <zephyr/net/net_if.h>
#include <zephyr/net/wifi.h>
#include <zephyr/net/wifi_mgmt.h>
#include <zephyr/net/net_event.h>
#include <zephyr/drivers/gpio.h>

#include <qspi_if.h>

#include <zephyr/net/socket.h>
#include <zephyr/net/mqtt.h>
#include <dk_buttons_and_leds.h>
#include <zephyr/random/rand32.h>

#define WIFI_SHELL_MODULE "wifi"

#define WIFI_SHELL_MGMT_EVENTS (NET_EVENT_WIFI_CONNECT_RESULT |		\
				NET_EVENT_WIFI_DISCONNECT_RESULT)

#define MAX_SSID_LEN        32
#define DHCP_TIMEOUT        70
#define CONNECTION_TIMEOUT  100
#define STATUS_POLLING_MS   300

/* 1000 msec = 1 sec */
#define LED_SLEEP_TIME_MS   100

/* The devicetree node identifier for the "led0" alias. */
#define LED0_NODE DT_ALIAS(led0)

LOG_MODULE_REGISTER(MQTT_OVER_WIFI, LOG_LEVEL_INF);

/* The mqtt client struct */
static struct mqtt_client client;
/* File descriptor */
static struct pollfd fds;

#define RANDOM_LEN 10
#define CLIENT_ID_LEN sizeof(CONFIG_BOARD) + 1 + RANDOM_LEN

/* Buffers for MQTT client. */
static uint8_t rx_buffer[CONFIG_MQTT_MESSAGE_BUFFER_SIZE];
static uint8_t tx_buffer[CONFIG_MQTT_MESSAGE_BUFFER_SIZE];
static uint8_t payload_buf[CONFIG_MQTT_PAYLOAD_BUFFER_SIZE];

/* MQTT Broker details. */
static struct sockaddr_storage broker;

/**@brief Function to get the payload of recived data.
 */
static int get_received_payload(struct mqtt_client *c, size_t length)
{
    int ret;
    int err = 0;

    /* Return an error if the payload is larger than the payload buffer.
     * Note: To allow new messages, we have to read the payload before returning.
     */
    if (length > sizeof(payload_buf)) {
        err = -EMSGSIZE;
    }

    /* Truncate payload until it fits in the payload buffer. */
    while (length > sizeof(payload_buf)) {
        ret = mqtt_read_publish_payload_blocking(
                c, payload_buf, (length - sizeof(payload_buf)));
        if (ret == 0) {
            return -EIO;
        } else if (ret < 0) {
            return ret;
        }

        length -= ret;
    }

    ret = mqtt_readall_publish_payload(c, payload_buf, length);
    if (ret) {
        return ret;
    }

    return err;
}

/**@brief Function to subscribe to the configured topic
 */

static int subscribe(struct mqtt_client *const c)
{
    struct mqtt_topic subscribe_topic = {
        .topic = {
            .utf8 = CONFIG_MQTT_SUB_TOPIC,
            .size = strlen(CONFIG_MQTT_SUB_TOPIC)
        },
        .qos = MQTT_QOS_1_AT_LEAST_ONCE
    };

    const struct mqtt_subscription_list subscription_list = {
        .list = &subscribe_topic,
        .list_count = 1,
        .message_id = 1234
    };

    LOG_INF("Subscribing to: %s len %u", CONFIG_MQTT_SUB_TOPIC,
        (unsigned int)strlen(CONFIG_MQTT_SUB_TOPIC));

    return mqtt_subscribe(c, &subscription_list);
}

/**@brief Function to print strings without null-termination
 */
static void data_print(uint8_t *prefix, uint8_t *data, size_t len)
{
    char buf[len + 1];

    memcpy(buf, data, len);
    buf[len] = 0;
    LOG_INF("%s%s", (char *)prefix, (char *)buf);
}

/**@brief Function to publish data on the configured topic
 */
int data_publish(struct mqtt_client *c, enum mqtt_qos qos,
    uint8_t *data, size_t len)
{
    struct mqtt_publish_param param;

    param.message.topic.qos = qos;
    param.message.topic.topic.utf8 = CONFIG_MQTT_PUB_TOPIC;
    param.message.topic.topic.size = strlen(CONFIG_MQTT_PUB_TOPIC);
    param.message.payload.data = data;
    param.message.payload.len = len;
    param.message_id = sys_rand32_get();
    param.dup_flag = 0;
    param.retain_flag = 0;

    data_print("Publishing: ", data, len);
    LOG_INF("to topic: %s len: %u",
        CONFIG_MQTT_PUB_TOPIC,
        (unsigned int)strlen(CONFIG_MQTT_PUB_TOPIC));

    return mqtt_publish(c, &param);
}
/**@brief MQTT client event handler
 */
void mqtt_evt_handler(struct mqtt_client *const c,
              const struct mqtt_evt *evt)
{
    int err;

    switch (evt->type) {
    case MQTT_EVT_CONNACK:
        if (evt->result != 0) {
            LOG_ERR("MQTT connect failed: %d", evt->result);
            break;
        }

        LOG_INF("MQTT client connected");
        subscribe(c);
        break;

    case MQTT_EVT_DISCONNECT:
        LOG_INF("MQTT client disconnected: %d", evt->result);
        break;

    case MQTT_EVT_PUBLISH:
    {
        const struct mqtt_publish_param *p = &evt->param.publish;
        //Print the length of the recived message
        LOG_INF("MQTT PUBLISH result=%d len=%d",
            evt->result, p->message.payload.len);

        //Extract the data of the recived message
        err = get_received_payload(c, p->message.payload.len);
        
        //Send acknowledgment to the broker on receiving QoS1 publish message
        if (p->message.topic.qos == MQTT_QOS_1_AT_LEAST_ONCE) {
            const struct mqtt_puback_param ack = {
                .message_id = p->message_id
            };

            /* Send acknowledgment. */
            mqtt_publish_qos1_ack(c, &ack);
        }

        if (err >= 0) {
            data_print("Received: ", payload_buf, p->message.payload.len);
            // Control LED1 and LED2
            if(strncmp(payload_buf,CONFIG_TURN_LED1_ON_CMD,sizeof(CONFIG_TURN_LED1_ON_CMD)-1) == 0){
                dk_set_led_on(DK_LED1);
            }
            else if(strncmp(payload_buf,CONFIG_TURN_LED1_OFF_CMD,sizeof(CONFIG_TURN_LED1_OFF_CMD)-1) == 0){
                dk_set_led_off(DK_LED1);
            }
            else if(strncmp(payload_buf,CONFIG_TURN_LED2_ON_CMD,sizeof(CONFIG_TURN_LED2_ON_CMD)-1) == 0){
                dk_set_led_on(DK_LED2);
            }
            else if(strncmp(payload_buf,CONFIG_TURN_LED2_OFF_CMD,sizeof(CONFIG_TURN_LED2_OFF_CMD)-1) == 0){
                dk_set_led_off(DK_LED2);
            }

        // Payload buffer is smaller than the received data
        } else if (err == -EMSGSIZE) {
            LOG_ERR("Received payload (%d bytes) is larger than the payload buffer size (%d bytes).",
                p->message.payload.len, sizeof(payload_buf));
        // Failed to extract data, disconnect
        } else {
            LOG_ERR("get_received_payload failed: %d", err);
            LOG_INF("Disconnecting MQTT client...");

            err = mqtt_disconnect(c);
            if (err) {
                LOG_ERR("Could not disconnect: %d", err);
            }
        }
    } break;

    case MQTT_EVT_PUBACK:
        if (evt->result != 0) {
            LOG_ERR("MQTT PUBACK error: %d", evt->result);
            break;
        }

        LOG_INF("PUBACK packet id: %u", evt->param.puback.message_id);
        break;

    case MQTT_EVT_SUBACK:
        if (evt->result != 0) {
            LOG_ERR("MQTT SUBACK error: %d", evt->result);
            break;
        }

        LOG_INF("SUBACK packet id: %u", evt->param.suback.message_id);
        break;

    case MQTT_EVT_PINGRESP:
        if (evt->result != 0) {
            LOG_ERR("MQTT PINGRESP error: %d", evt->result);
        }
        break;

    default:
        LOG_INF("Unhandled MQTT event type: %d", evt->type);
        break;
    }
}

/**@brief Resolves the configured hostname and
 * initializes the MQTT broker structure
 */
static int broker_init(void)
{
    int err;
    struct addrinfo *result;
    struct addrinfo *addr;
    struct addrinfo hints = {
        .ai_family = AF_INET,
        .ai_socktype = SOCK_STREAM
    };
 
    err = getaddrinfo("91.121.93.94", NULL, &hints, &result);
//    err = getaddrinfo("test.mosquitto.org", NULL, &hints, &result);
    if (err) {
        LOG_ERR("getaddrinfo failed: %d", err);
        return -ECHILD;
    }

    addr = result;

    /* Look for address of the broker. */
    while (addr != NULL) {
        /* IPv4 Address. */
        if (addr->ai_addrlen == sizeof(struct sockaddr_in)) {
            struct sockaddr_in *broker4 =
                ((struct sockaddr_in *)&broker);
            char ipv4_addr[NET_IPV4_ADDR_LEN];

            broker4->sin_addr.s_addr =
                ((struct sockaddr_in *)addr->ai_addr)
                ->sin_addr.s_addr;
            broker4->sin_family = AF_INET;
            broker4->sin_port = htons(CONFIG_MQTT_BROKER_PORT);

            inet_ntop(AF_INET, &broker4->sin_addr.s_addr,
                  ipv4_addr, sizeof(ipv4_addr));
            LOG_INF("IPv4 Address found %s", (char *)(ipv4_addr));

            break;
        } else {
            LOG_ERR("ai_addrlen = %u should be %u or %u",
                (unsigned int)addr->ai_addrlen,
                (unsigned int)sizeof(struct sockaddr_in),
                (unsigned int)sizeof(struct sockaddr_in6));
        }

        addr = addr->ai_next;
    }

    /* Free the address. */
    freeaddrinfo(result);

    return err;
}

/* Function to get the client id */
static const uint8_t* client_id_get(void)
{
    static uint8_t client_id[MAX(sizeof(CONFIG_MQTT_CLIENT_ID),
                     CLIENT_ID_LEN)];

    if (strlen(CONFIG_MQTT_CLIENT_ID) > 0) {
        snprintf(client_id, sizeof(client_id), "%s",
             CONFIG_MQTT_CLIENT_ID);
        goto exit;
    }

    uint32_t id = sys_rand32_get();
    snprintf(client_id, sizeof(client_id), "%s-%010u", CONFIG_BOARD, id);

exit:
    LOG_DBG("client_id = %s", (char *)client_id);

    return client_id;
}


/**@brief Initialize the MQTT client structure
 */
int client_init(struct mqtt_client *client)
{
    int err;
    /* Initializes the client instance. */
    mqtt_client_init(client);

    /* Resolves the configured hostname and initializes the MQTT broker structure */
    err = broker_init();
    if (err) {
        LOG_ERR("Failed to initialize broker connection");
        return err;
    }

    /* MQTT client configuration */
    client->broker = &broker;
    client->evt_cb = mqtt_evt_handler;
    client->client_id.utf8 = client_id_get();
    client->client_id.size = strlen(client->client_id.utf8);
    client->password = NULL;
    client->user_name = NULL;
    client->protocol_version = MQTT_VERSION_3_1_1;

    /* MQTT buffers configuration */
    client->rx_buf = rx_buffer;
    client->rx_buf_size = sizeof(rx_buffer);
    client->tx_buf = tx_buffer;
    client->tx_buf_size = sizeof(tx_buffer);

    /* We are not using TLS */
    client->transport.type = MQTT_TRANSPORT_NON_SECURE;


    return err;
}

/**@brief Initialize the file descriptor structure used by poll.
 */
int fds_init(struct mqtt_client *c, struct pollfd *fds)
{
    if (c->transport.type == MQTT_TRANSPORT_NON_SECURE) {
        fds->fd = c->transport.tcp.sock;
    } else {
        return -ENOTSUP;
    }

    fds->events = POLLIN;

    return 0;
}

/*
 * A build error on this line means your board is unsupported.
 * See the sample documentation for information on how to fix this.
 */
//static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

static struct net_mgmt_event_callback wifi_shell_mgmt_cb;
static struct net_mgmt_event_callback net_shell_mgmt_cb;

static struct {
	const struct shell *sh;
	union {
		struct {
			uint8_t connected	: 1;
			uint8_t connect_result	: 1;
			uint8_t disconnect_requested	: 1;
			uint8_t _unused		: 5;
		};
		uint8_t all;
	};
} context;


static int cmd_wifi_status(void)
{
	struct net_if *iface = net_if_get_default();
	struct wifi_iface_status status = { 0 };

	if (net_mgmt(NET_REQUEST_WIFI_IFACE_STATUS, iface, &status,
				sizeof(struct wifi_iface_status))) {
		LOG_INF("Status request failed");

		return -ENOEXEC;
	}

	LOG_INF("==================");
	LOG_INF("State: %s", wifi_state_txt(status.state));

	if (status.state >= WIFI_STATE_ASSOCIATED) {
		uint8_t mac_string_buf[sizeof("xx:xx:xx:xx:xx:xx")];

		LOG_INF("Interface Mode: %s",
		       wifi_mode_txt(status.iface_mode));
		LOG_INF("Link Mode: %s",
		       wifi_link_mode_txt(status.link_mode));
		LOG_INF("SSID: %-32s", status.ssid);
		LOG_INF("BSSID: %s",
		       net_sprint_ll_addr_buf(
				status.bssid, WIFI_MAC_ADDR_LEN,
				mac_string_buf, sizeof(mac_string_buf)));
		LOG_INF("Band: %s", wifi_band_txt(status.band));
		LOG_INF("Channel: %d", status.channel);
		LOG_INF("Security: %s", wifi_security_txt(status.security));
		LOG_INF("MFP: %s", wifi_mfp_txt(status.mfp));
		LOG_INF("RSSI: %d", status.rssi);
	}
	return 0;
}

static void handle_wifi_connect_result(struct net_mgmt_event_callback *cb)
{
	const struct wifi_status *status =
		(const struct wifi_status *) cb->info;

	if (context.connected) {
		return;
	}

	if (status->status) {
		LOG_ERR("Connection failed (%d)", status->status);
	} else {
		LOG_INF("Connected");
		context.connected = true;
	}
	context.connect_result = true;
}

static void handle_wifi_disconnect_result(struct net_mgmt_event_callback *cb)
{
	const struct wifi_status *status =
		(const struct wifi_status *) cb->info;

	if (!context.connected) {
		return;
	}

	if (context.disconnect_requested) {
		LOG_INF("Disconnection request %s (%d)",
			 status->status ? "failed" : "done",
					status->status);
		context.disconnect_requested = false;
	} else {
		LOG_INF("Received Disconnected");
		context.connected = false;
	}
	cmd_wifi_status();
}

static void wifi_mgmt_event_handler(struct net_mgmt_event_callback *cb,
				     uint32_t mgmt_event, struct net_if *iface)
{
	switch (mgmt_event) {
	case NET_EVENT_WIFI_CONNECT_RESULT:
		handle_wifi_connect_result(cb);
		break;
	case NET_EVENT_WIFI_DISCONNECT_RESULT:
		handle_wifi_disconnect_result(cb);
		break;
	default:
		break;
	}
}

static void print_dhcp_ip(struct net_mgmt_event_callback *cb)
{
	/* Get DHCP info from struct net_if_dhcpv4 and print */
	const struct net_if_dhcpv4 *dhcpv4 = cb->info;
	const struct in_addr *addr = &dhcpv4->requested_ip;
	char dhcp_info[128];

	net_addr_ntop(AF_INET, addr, dhcp_info, sizeof(dhcp_info));

	LOG_INF("DHCP IP address: %s", dhcp_info);
}
static void net_mgmt_event_handler(struct net_mgmt_event_callback *cb,
				    uint32_t mgmt_event, struct net_if *iface)
{
	switch (mgmt_event) {
	case NET_EVENT_IPV4_DHCP_BOUND:
		print_dhcp_ip(cb);
		break;
	default:
		break;
	}
}

static int __wifi_args_to_params(struct wifi_connect_req_params *params)
{
	params->timeout = SYS_FOREVER_MS;

	/* SSID */
	params->ssid = CONFIG_STA_SAMPLE_SSID;
	params->ssid_length = strlen(params->ssid);

#if defined(CONFIG_STA_KEY_MGMT_WPA2)
	params->security = 1;
#elif defined(CONFIG_STA_KEY_MGMT_WPA2_256)
	params->security = 2;
#elif defined(CONFIG_STA_KEY_MGMT_WPA3)
	params->security = 3;
#else
	params->security = 0;
#endif

#if !defined(CONFIG_STA_KEY_MGMT_NONE)
	params->psk = CONFIG_STA_SAMPLE_PASSWORD;
	params->psk_length = strlen(params->psk);
#endif
	params->channel = WIFI_CHANNEL_ANY;

	/* MFP (optional) */
	params->mfp = WIFI_MFP_OPTIONAL;

	return 0;
}

static int wifi_connect(void)
{
	struct net_if *iface = net_if_get_default();
	static struct wifi_connect_req_params cnx_params;

	context.connected = false;
	context.connect_result = false;
	__wifi_args_to_params(&cnx_params);

	if (net_mgmt(NET_REQUEST_WIFI_CONNECT, iface,
		     &cnx_params, sizeof(struct wifi_connect_req_params))) {
		LOG_ERR("Connection request failed");

		return -ENOEXEC;
	}

	LOG_INF("Connection requested");

	return 0;
}

int bytes_from_str(const char *str, uint8_t *bytes, size_t bytes_len)
{
	size_t i;
	char byte_str[3];

	if (strlen(str) != bytes_len * 2) {
		LOG_ERR("Invalid string length: %zu (expected: %d)\n",
			strlen(str), bytes_len * 2);
		return -EINVAL;
	}

	for (i = 0; i < bytes_len; i++) {
		memcpy(byte_str, str + i * 2, 2);
		byte_str[2] = '\0';
		bytes[i] = strtol(byte_str, NULL, 16);
	}

	return 0;
}


static void button_handler(uint32_t button_state, uint32_t has_changed)
{
    switch (has_changed) {
    case DK_BTN1_MSK:
        if (button_state & DK_BTN1_MSK){
            int err = data_publish(&client, MQTT_QOS_1_AT_LEAST_ONCE,
                   CONFIG_BUTTON1_EVENT_PUBLISH_MSG, sizeof(CONFIG_BUTTON1_EVENT_PUBLISH_MSG)-1);
            if (err) {
                LOG_ERR("Failed to send message, %d", err);
                return;
            }
        }
        break;
    case DK_BTN2_MSK:
        if (button_state & DK_BTN2_MSK){
            int err = data_publish(&client, MQTT_QOS_1_AT_LEAST_ONCE,
                   CONFIG_BUTTON2_EVENT_PUBLISH_MSG, sizeof(CONFIG_BUTTON2_EVENT_PUBLISH_MSG)-1);
            if (err) {
                LOG_ERR("Failed to send message, %d", err);
                return;
            }
        }
        break;
    }
}

static void connect_mqtt(void)
{
    int err;
    uint32_t connect_attempt = 0;

    if (dk_leds_init() != 0) {
        LOG_ERR("Failed to initialize the LED library");
    }

    if (dk_buttons_init(button_handler) != 0) {
        LOG_ERR("Failed to initialize the buttons library");
    }

    err = client_init(&client);
    if (err) {
        LOG_ERR("Failed to initialize MQTT client: %d", err);
        return;
    }

do_connect:
    if (connect_attempt++ > 0) {
        LOG_INF("Reconnecting in %d seconds...",
            CONFIG_MQTT_RECONNECT_DELAY_S);
        k_sleep(K_SECONDS(CONFIG_MQTT_RECONNECT_DELAY_S));
    }
    err = mqtt_connect(&client);
    if (err) {
        LOG_ERR("Error in mqtt_connect: %d", err);
        goto do_connect;
    }

    err = fds_init(&client,&fds);
    if (err) {
        LOG_ERR("Error in fds_init: %d", err);
        return;
    }

    while (1) {
        err = poll(&fds, 1, mqtt_keepalive_time_left(&client));
        if (err < 0) {
            LOG_ERR("Error in poll(): %d", errno);
            break;
        }

        err = mqtt_live(&client);
        if ((err != 0) && (err != -EAGAIN)) {
            LOG_ERR("Error in mqtt_live: %d", err);
            break;
        }

        if ((fds.revents & POLLIN) == POLLIN) {
            err = mqtt_input(&client);
            if (err != 0) {
                LOG_ERR("Error in mqtt_input: %d", err);
                break;
            }
        }

        if ((fds.revents & POLLERR) == POLLERR) {
            LOG_ERR("POLLERR");
            break;
        }

        if ((fds.revents & POLLNVAL) == POLLNVAL) {
            LOG_ERR("POLLNVAL");
            break;
        }
    }

    LOG_INF("Disconnecting MQTT client");

    err = mqtt_disconnect(&client);
    if (err) {
        LOG_ERR("Could not disconnect MQTT client: %d", err);
    }
    goto do_connect;
}


static void wifi_config_init(void) {
    
    memset(&context, 0, sizeof(context));

    net_mgmt_init_event_callback(&wifi_shell_mgmt_cb,
                     wifi_mgmt_event_handler,
                     WIFI_SHELL_MGMT_EVENTS);

    net_mgmt_add_event_callback(&wifi_shell_mgmt_cb);


    net_mgmt_init_event_callback(&net_shell_mgmt_cb,
                     net_mgmt_event_handler,
                     NET_EVENT_IPV4_DHCP_BOUND);

    net_mgmt_add_event_callback(&net_shell_mgmt_cb);
}


void main(void)
{
    /* Wait for the NET_EVENT_WIFI_CONNECT_RESULT event*/
    int i;
    wifi_config_init();
    
    LOG_INF("Starting %s with CPU frequency: %d MHz", CONFIG_BOARD, SystemCoreClock/MHZ(1));
    k_sleep(K_SECONDS(1));

#ifdef CONFIG_BOARD_NRF7002DK_NRF5340
   if (strlen(CONFIG_NRF700X_QSPI_ENCRYPTION_KEY)) {
       char key[QSPI_KEY_LEN_BYTES];
       int ret;

       ret = bytes_from_str(CONFIG_NRF700X_QSPI_ENCRYPTION_KEY, key, sizeof(key));
       if (ret) {
           LOG_ERR("Failed to parse encryption key: %d\n", ret);
           return;
       }

       LOG_DBG("QSPI Encryption key: ");
       for (int i = 0; i < QSPI_KEY_LEN_BYTES; i++) {
           LOG_DBG("%02x", key[i]);
       }
       LOG_DBG("\n");

       ret = qspi_enable_encryption(key);
       if (ret) {
           LOG_ERR("Failed to enable encryption: %d\n", ret);
           return;
       }
       LOG_INF("QSPI Encryption enabled");
   } else {
       LOG_INF("QSPI Encryption disabled");
   }
#endif

    LOG_INF("Static IP address (overridable): %s/%s -> %s",
        CONFIG_NET_CONFIG_MY_IPV4_ADDR,
        CONFIG_NET_CONFIG_MY_IPV4_NETMASK,
        CONFIG_NET_CONFIG_MY_IPV4_GW);

    /* Wait for the interface to be up */
    k_sleep(K_SECONDS(5));

    while (1) {

        wifi_connect();

        for (i = 0; i < CONNECTION_TIMEOUT; i++) {
            k_sleep(K_MSEC(STATUS_POLLING_MS));
            cmd_wifi_status();
            if (context.connect_result) {
                break;
            }
        }
        if (context.connected) {
            break;
        } else if (!context.connect_result) {
            LOG_ERR("Connection Timed Out");
            //k_sem_take(&wifi_connected_sem, K_FOREVER);
        }
    }

    k_sleep(K_SECONDS(5));
    /* Connect to MQTT Broker */
    LOG_INF("Connecting to MQTT Broker... \n");
    connect_mqtt();
}
```


```prj.conf
#
# Copyright (c) 2022 Nordic Semiconductor ASA
#
# SPDX-License-Identifier: LicenseRef-Nordic-5-Clause
#
CONFIG_WIFI=y
CONFIG_WIFI_NRF700X=y

# WPA supplicant
CONFIG_WPA_SUPP=y

# System settings
CONFIG_NEWLIB_LIBC=y
CONFIG_NEWLIB_LIBC_NANO=n

# Networking
CONFIG_NETWORKING=y
CONFIG_NET_SOCKETS=y
CONFIG_POSIX_CLOCK=y
CONFIG_NET_SOCKETS_POSIX_NAMES=y
CONFIG_NET_LOG=y
CONFIG_NET_IPV6=n
CONFIG_NET_IPV4=y
CONFIG_NET_UDP=y
CONFIG_NET_TCP=y
CONFIG_NET_DHCPV4=y
CONFIG_DNS_RESOLVER=y
CONFIG_NET_NATIVE=y

# CONFIG_STA_KEY_MGMT_NONE=y
# CONFIG_STA_KEY_MGMT_WPA2=y
# CONFIG_STA_KEY_MGMT_WPA2_256=y
# CONFIG_STA_KEY_MGMT_WPA3=y
CONFIG_STA_SAMPLE_SSID="MySSID"
CONFIG_STA_SAMPLE_PASSWORD="MySSID-Password"

CONFIG_NET_STATISTICS=y

CONFIG_NET_PKT_RX_COUNT=8
CONFIG_NET_PKT_TX_COUNT=8

# Below section is the primary contributor to SRAM and is currently
# tuned for performance, but this will be revisited in the future.
CONFIG_NET_BUF_RX_COUNT=16
CONFIG_NET_BUF_TX_COUNT=16
CONFIG_NET_BUF_DATA_SIZE=512
CONFIG_HEAP_MEM_POOL_SIZE=153600
CONFIG_NET_TC_TX_COUNT=0

CONFIG_NET_IF_UNICAST_IPV4_ADDR_COUNT=1
CONFIG_NET_MAX_CONTEXTS=5
CONFIG_NET_CONTEXT_SYNC_RECV=y

CONFIG_INIT_STACKS=y

CONFIG_NET_L2_ETHERNET=y

CONFIG_NET_CONFIG_SETTINGS=y

CONFIG_NET_CONFIG_MY_IPV4_ADDR="192.165.1.100"
CONFIG_NET_CONFIG_PEER_IPV4_ADDR="192.165.1.1"

CONFIG_NET_SOCKETS_POLL_MAX=4

# Memories
CONFIG_MAIN_STACK_SIZE=4096
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
CONFIG_NET_TX_STACK_SIZE=4096
CONFIG_NET_RX_STACK_SIZE=4096

# Debugging
CONFIG_STACK_SENTINEL=y
CONFIG_DEBUG_COREDUMP=y
CONFIG_DEBUG_COREDUMP_BACKEND_LOGGING=y
CONFIG_DEBUG_COREDUMP_MEMORY_DUMP_MIN=y

# Kernel options
CONFIG_ENTROPY_GENERATOR=y
CONFIG_NET_L2_WIFI_MGMT=y
CONFIG_NET_MGMT_EVENT_STACK_SIZE=2048
CONFIG_FLASH=y
CONFIG_FLASH_PAGE_LAYOUT=y
CONFIG_FLASH_MAP=y
CONFIG_NVS=y
CONFIG_SETTINGS=y
CONFIG_SETTINGS_NVS=y

CONFIG_NANOPB=y

# Logging
CONFIG_LOG=y

# Button and LED support
CONFIG_DK_LIBRARY=y

# MQTT
CONFIG_MQTT_LIB=y
CONFIG_MQTT_CLEAN_SESSION=y

# Application
CONFIG_MQTT_PUB_TOPIC="devacademy/publish/topic44"
CONFIG_MQTT_SUB_TOPIC="devacademy/subscribe/topic44"
CONFIG_MQTT_BROKER_HOSTNAME="test.mosquitto.org"
CONFIG_MQTT_CLIENT_ID="nRF7002DK_Test_Ali"
CONFIG_MQTT_BROKER_PORT=1883
```






```Kconfig
#
# Copyright (c) 2022 Nordic Semiconductor
#
# SPDX-License-Identifier: LicenseRef-Nordic-5-Clause
#
Kconfig.template.log_config"
source "Kconfig.zephyr"

menu "Nordic Sta sample"

config MQTT_PUB_TOPIC
	string "MQTT publish topic"
	default "devacademy/publish/topic44"

config MQTT_SUB_TOPIC
	string "MQTT subscribe topic"
	default "devacademy/subscribe/topic44"

config MQTT_CLIENT_ID
	string "MQTT Client ID"
	help
	  Use a custom Client ID string.
	default "nRF7002DK_Test_Ali"

config MQTT_BROKER_HOSTNAME
	string "MQTT broker hostname"
	default "test.mosquitto.org"

config MQTT_BROKER_PORT
	int "MQTT broker port"
	default 1883

config MQTT_MESSAGE_BUFFER_SIZE
	int "MQTT message buffer size"
	default 128

config MQTT_PAYLOAD_BUFFER_SIZE
	int "MQTT payload buffer size"
	default 128

config BUTTON1_EVENT_PUBLISH_MSG
	string "The message to publish on a button event"
	default "Button 1 Pressed"

config BUTTON2_EVENT_PUBLISH_MSG
	string "The message to publish on a button event"
	default "Button 2 Pressed"

config TURN_LED1_ON_CMD
	string "Command to turn on LED"
	default "LED1ON"

config TURN_LED1_OFF_CMD
	string "Command to turn off LED"
	default "LED1OFF"

config TURN_LED2_ON_CMD
	string "Command to turn on LED"
	default "LED2ON"

config TURN_LED2_OFF_CMD
	string "Command to turn off LED"
	default "LED2OFF"

config MQTT_RECONNECT_DELAY_S
	int "Seconds to delay before attempting to reconnect to the broker."
	default 60

config CONNECTION_IDLE_TIMEOUT
	int "Time to be waited for a station to connect"
	default 30

config STA_SAMPLE_SSID
	string "SSID"
	help
	  Specify the SSID to connect

choice  STA_KEY_MGMT_SELECT
	prompt "Security Option"
	default STA_KEY_MGMT_WPA3

config STA_KEY_MGMT_NONE
	bool "Open Security"
	help
	  Enable for Open Security

config STA_KEY_MGMT_WPA2
	bool "WPA2 Security"
	help
	  Enable for WPA2 Security

config STA_KEY_MGMT_WPA2_256
	bool "WPA2 SHA 256 Security"
	help
	  Enable for WPA2-PSK-256 Security

config STA_KEY_MGMT_WPA3
	bool "WPA3 Security"
	help
	  Enable for WPA3 Security
endchoice

config STA_SAMPLE_PASSWORD
	string "Passphrase (WPA2) or password (WPA3)"
	help
	  Specify the Password to connect

config NRF700X_QSPI_ENCRYPTION_KEY
	string "16 bytes QSPI encryption key, only for testing purposes"
	depends on BOARD_NRF7002DK_NRF5340
	help
	  Specify the QSPI encryption key
endmenu
```

