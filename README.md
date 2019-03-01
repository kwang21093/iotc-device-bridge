# Fork Changes for Azure Iot Central Device Bridge to represent bridge as a gateway device within Azure Iot Central
Current fork has changes to enable Azure Iot Central Device Bridge to be presented in IotCental as a gateway.

## New Azure Iot Central features to display generic gateway devices  
With latest released changes of Azure Iot Central users are able to see parent child relationship between generic gateway device and child devices connected through a given gateway. 

During device registartion device can reigester itself to have following characteristics:
- Regular device connected directly to IotCentral
- Generic gateway device
- Device connected to IotCentral through generic gateway. Device indicates its parent gateway device Id through which it is connected to.


### Displaying device gateway information within device explorer ui and device sets
You can see gateway connection attributtes within device explorer or select corresponding columns when defining deviceset. 

Unassosiated gateway device
![Gateway device](assets\UnassociatedGateway.png "Gateway device")  

Unassociated Device connected to gateway
![Unassociated Device connected to gateway](assets\UnassociatedGateway.png "Unassociated Device connected to gateway") 

Associated devices view
![Device Explorer with gateway information](assets\AssociatedGatewayAndChild.png "Device Explorer with gateway information")

Configure deviceset to display gateway related columns

![Configure deviceset to display gateway related columns](assets\gateway-deviceset-configure.png "Configure deviceset to display gateway related columns")

### Search to display all devices connected to a given gateway
Full text search has been extended to search within deviceGatewayId represented as 'Connected through' column in device explorer.
You can see all child devices connected to a given gateway by entering gateway device id within search input.

![Search to display all devices connected to a given gateway ](assets\gateway-search.png "Search to display all devices connected to a given gateway")

## What has been changed in fork
See original documentation in Read.me of upstream branch for complete deployment instruction.

[azuredeploy.json](azuredeploy.json) has been modified to include 2 new parameters:

"iotCentralActAsGateway": {
            "type": "bool"
        },
        "iotCentralGatewayDeviceId": {
            "type": "String"
}
 Act as Gateway is a boolean flag indicating if device bridge should register itself as a gateway within IotCentral. Set this flag to true otherwise solution will work as upstream branch.

 Gateway Device Id - is device id which will represent bridge as a gateway device within Azure Iot Central 

 ![Act as Gateway and gateway device id](assets\DeploymentArmChanges.png "Act as Gateway and gateway device id")


## [lib/engine.js changes](IoTCIntegration\lib\engine.js)

A code block has been introduce to register bridge as gateway when azure function receives it's first request. Please take note that none of subsequent request will be proccessed until bridge will be assocciated with any device model in iot central app

if (context.actAsGateway) {
        if (!gatewayDevice) {
            gatewayDevice = { deviceId: context.gatewayDeviceId };
        }

        const gatewayClient = Device.Client.fromConnectionString(await getDeviceConnectionString(context, gatewayDevice), DeviceTransport.Http);
        try {
            await util.promisify(gatewayClient.open.bind(gatewayClient))();
            context.log('[HTTP] Sending telemetry for gateway device', gatewayDevice.deviceId);
            // TODO: add any gateway specific telemetry if needed
            // await util.promisify(gatewayClient.sendEvent.bind(gatewayClient))(new Device.Message(JSON.stringify({["ping"]:1})));
            await util.promisify(gatewayClient.close.bind(gatewayClient))();

        } catch (e) {
            // If the device was deleted, we remove its cached connection string
            if (e.name === 'DeviceNotFoundError' && deviceCache[gatewayDevice.deviceId]) {
                delete deviceCache[gatewayDevice.deviceId].connectionString;
            }
            throw new Error(`Unable to send telemetry for gateway device ${gatewayDevice.deviceId}: ${e.message}`);
        }

        device.gatewayId = gatewayDevice.deviceId;
    }
</code>
  
Device Provisioning Service api call has been changed to add extra post body information to indicate if device is gateway device or child device connected to gateway.

const bodyJson = {
        registrationId: deviceId
    };

    if (context.actAsGateway) {
        if (device.gatewayId) {
            bodyJson["data"] = {
                iotcGateway: {
                    iotcGatewayId: device.gatewayId,
                    iotcIsGateway: false
                }
            }
        } else {
            bodyJson["data"] = {
                iotcGateway: {
                    iotcGatewayId: null,
                    iotcIsGateway: true
                }
            }
        }
    }

    const registrationOptions = {
        url: `https://${registrationHost}/${context.idScope}/registrations/${deviceId}/register?api-version=${registrationApiVersion}`,
        method: 'PUT',
        json: true,
        headers: { Authorization: sasToken },
        body: bodyJson,
    };