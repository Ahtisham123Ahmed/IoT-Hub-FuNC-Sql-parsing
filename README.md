# IoT-Hub-FuNC-Sql-parsing

index.js
const sql = require('mssql');

// SQL configuration – ensure credentials & server are correct
const sqlConfig = {
  user: 'Ahtisham',
  password: 'Leansol@498',
  server: 'your-sql-server.database.windows.net',
  database: 'EnergyMeterDB',
  options: {
    encrypt: true,
    trustServerCertificate: false
  }
};

module.exports = async function (context, IoTHubMessages) {
  context.log(`JavaScript eventhub trigger function called for ${IoTHubMessages.length} messages`);
  
  try {
    // Connect to SQL
    await sql.connect(sqlConfig);
    context.log("Connected to SQL database");
    
    // Prepare a reusable statement
    const ps = new sql.PreparedStatement();
    ps.input('messageId', sql.Int);
    ps.input('deviceId', sql.NVarChar(100));
    ps.input('temperature', sql.Float);
    ps.input('humidity', sql.Float);
    
    await ps.prepare(`
      INSERT INTO dbo.SensorData 
        (MessageId, DeviceId, Temperature, Humidity, Timestamp)
      VALUES (@messageId, @deviceId, @temperature, @humidity, GETUTCDATE())
    `);

    // Loop through each incoming message
    for (const message of IoTHubMessages) {
      try {
        // Parse JSON if it's a string
        const msg = typeof message === 'string' ? JSON.parse(message) : message;

        // Support message format either flat or nested inside .body
        const src = msg.body && typeof msg.body === 'object' ? msg.body : msg;

        context.log(`Processing message: ${JSON.stringify(src)}`);

        // Validate required fields
        if (
          src.messageId == null ||
          src.deviceId == null ||
          src.temperature == null ||
          src.humidity == null
        ) {
          throw new Error('Missing one or more required fields (messageId, deviceId, temperature, humidity)');
        }

        // Insert into SQL
        await ps.execute({
          messageId: src.messageId,
          deviceId: src.deviceId,
          temperature: src.temperature,
          humidity: src.humidity
        });

        context.log(`Inserted message ${src.messageId} into SQL`);
      } catch (innerErr) {
        context.log.error(`Error processing message: ${innerErr.message}`);
      }
    }

    await ps.unprepare();
  } catch (outerErr) {
    context.log.error(`SQL CONNECTION ERROR: ${outerErr.message}`);
  } finally {
    await sql.close();
    context.log("SQL connection closed");
  }
};

Function.JS


{
  "bindings": [
    {
      "type": "eventHubTrigger",
      "name": "IoTHubMessages",
      "direction": "in",
      "eventHubName": "samples-workitems",
      "connection": "Energy-Meter-Enfana_events_IOTHUB",
      "cardinality": "many",
      "consumerGroup": "$Default",
      "dataType": "string"
    }
  ]
}
SQL  Code
