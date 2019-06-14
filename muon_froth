import java.util.*;
import java.io.*;
import java.net.*;

import processing.serial.*;

import processing.net.*;

int httpTimeout = 3000;
Display display;

Serial muonPort;

Vector<MuonEvent> sensorEvents = new Vector<MuonEvent>();

//the min and max amount of time to fade the bulbs
int minFade = 1000;
int maxFade = 3000;

int minFrothTime = 1000;
int maxFrothTime = 6000;

int muonX_draw = 0;
int muonY_draw = 0;
boolean draw = false;
int drawScale = width/20;


class MuonEvent {
  public int x;
  public int y;
  public long time;
  public long frothTime;

  public MuonEvent(int x, int y, long frothTime) {
    this.x = x;
    this.y = y;
    this.frothTime = frothTime;
    this.time = millis();
  }
}

void setup() {

  size(260, 360);
  background(255);
  printArray(Serial.list());
  String portName = Serial.list()[0];

  muonPort = new Serial(this, portName, 115200);

  display = new Display(18, 10, 15, loadStrings("bulbs.csv"));
  Thread displayThread = new Thread(display);
  displayThread.start();

  Thread muonThread = new Thread(new MuonTracker(muonPort));
  muonThread.start();
}

void draw() {
  if (draw == true) {

    drawMuon();
    draw = false;
  }
}

void mouseClicked() {
  background(255);

  int muonX = (int)map(mouseX, 0, width, 1, 6);
  int muonY = (int)map(mouseY, 0, height, 1, 7);

  updateOLED(muonX, muonY);

  muonX = (int)map(muonX, 1, 5, 1, 10);
  muonY = (int)map(muonY, 1, 6, 1, 18);
  fill(0);
  ellipse(mouseX, mouseY, drawScale, drawScale);

  int frothy = (int)(minFrothTime + random(maxFrothTime));
  MuonEvent muonEvent = new MuonEvent(muonX, muonY, frothy);
  synchronized (sensorEvents) {
    sensorEvents.add(muonEvent);
  }
  println("Simulated muon event");
  println("location: " + muonX + "," + muonY);
  frothy = frothy/1000;
  println("overall fade time: " + frothy + " seconds");
}

class BulbConnection {
  private String hostname;

  public BulbConnection(String hostname) {
    this.hostname = hostname;
  }

  private String get(String path) throws IOException {
    URL url;

    try {
      url = new URL("http://" + this.hostname + path);
    } 
    catch (MalformedURLException e) {
      e.printStackTrace();
      return null;
    }
    //println(url);

    HttpURLConnection httpConnection = (HttpURLConnection)url.openConnection();
    httpConnection.setConnectTimeout(httpTimeout);
    httpConnection.setRequestMethod("GET");
    BufferedReader reader = new BufferedReader(new InputStreamReader(httpConnection.getInputStream()));

    StringBuilder result = new StringBuilder();

    String line;
    while ((line = reader.readLine()) != null) {
      result.append(line);
    }

    reader.close();

    return result.toString();
  }

  public void blink(float min, float max, int duration) throws IOException {
    int minValue = (int)map(constrain(min, 0, 1), 0, 1, 0, 1023);
    int maxValue = (int)map(constrain(max, 0, 1), 0, 1, 0, 1023);
    String path = "/blink?MIN=" + minValue + "&MAX=" + maxValue + "&DURATION=" + duration;
    //println(path);
    get(path);
  }

  public String hostname() throws IOException {
    return get("/hostname") + ".local";
  }
}



class Display implements Runnable {
  private Vector<String> unmappedIPs;
  private HashMap<String, String> ipsByName;
  private Vector<String> hostsInOrder;

  public Display(int width, int depth, int ledsPerStrand, String[] hostnames) {
    unmappedIPs = new Vector<String>();
    ipsByName = new HashMap<String, String>();
    hostsInOrder = new Vector<String>();

    hostsInOrder.addAll(Arrays.asList(hostnames));

    //loadHostInfo();
  }

  private String getIP(String hostname) {
    if (ipsByName.containsKey(hostname)) {
      return ipsByName.get(hostname);
    } else {
      return null;
    }
  }

  private void mapIPsToHostnames() {
    int oldTimeout = httpTimeout;
    httpTimeout = 1000;

    for (int i = 2; i < 256; i++) {
      unmappedIPs.add("192.168.0." + i);
    }

    while (unmappedIPs.size() > 0) {
      String nextIP = unmappedIPs.get(0);
      //println("Getting hostname for " + nextIP);

      BulbConnection connection = new BulbConnection(nextIP);

      try {
        String hostname = connection.hostname();
        //println("Hostname: " + hostname);
        println(nextIP + " -> " + hostname);
        ipsByName.put(hostname, nextIP);
      } 
      catch (IOException e) {
        //println("Failed to connect to " + nextIP);
        //e.printStackTrace();
      }

      unmappedIPs.remove(0);
    }

    httpTimeout = oldTimeout;

    saveHostInfo();
    println("Hostname mapping saved");
  }

  void saveHostInfo() {
    Table hostnamesTable = new Table();

    for (String hostname : hostsInOrder) {
      String ip = getIP(hostname);

      if (ip == null) {
        continue;
      }

      TableRow row = hostnamesTable.addRow();
      row.setString(0, hostname);
      row.setString(1, ip);
    }
    saveTable(hostnamesTable, "hostnames.csv");
  }

  void loadHostInfo() {
    Table hostnamesTable = loadTable("hostnames.csv");

    for (TableRow row : hostnamesTable.rows()) {
      String hostname = row.getString(0);
      String ip = row.getString(1);
      ipsByName.put(hostname, ip);
    }
  }

  void asyncBlink(final String host, final float min, final float max, final int duration) {
    (new Thread(new Runnable() {
      public void run() {
        BulbConnection bulb = new BulbConnection(host);

        try {
          bulb.blink(min, max, duration);
        } 
        catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    )).start();
  }

  String addressByXY(int x, int y) {
    boolean leftToRight = y % 2 == 0;
    int index = y * 10 + (leftToRight ? x : (10 - x - 1));

    return getIP(hostsInOrder.get(index));
  }

  void blinkXY(int x, int y, float min, float max, int duration) {
    String address = addressByXY(x, y);
    if (address == null) {
      //println("No host for " + x + ", " + y);
      return;
    }

    asyncBlink(address, min, max, duration);
  }

  void updateDisplay() {
    for (String host : hostsInOrder) {
      final String ip = getIP(host);

      if (ip == null) {
        continue;
      }

      asyncBlink(ip, 0, 1, 1000);
    }
  }

  private boolean hostnamesFileExists() {
    File hostnamesFile = dataFile("../hostnames.csv");
    return hostnamesFile.isFile();
  }

  void run() {
    if (hostnamesFileExists()) {
      loadHostInfo();
    } else {
      println("Hostname to IP mapping file does not exist, regenerating...");
      Table hostnamesTable = new Table();
      mapIPsToHostnames();
    }

    println("Controlling " + ipsByName.size() + " bulbs");

    // Turn on the entire grid
    for (int y = 0; y < 18; y++) {
      for (int x = 0; x < 10; x++) {
        blinkXY(x, y, 1, 1, 10);
        try {
          Thread.sleep(50);
        } 
        catch (InterruptedException e) {
        }
      }
    }

    int i = 0;
    boolean direction = false;
    while (true) {
      synchronized (sensorEvents) {
        Iterator<MuonEvent> eventItr = sensorEvents.iterator();
        while (eventItr.hasNext()) {
          MuonEvent event = eventItr.next();
          long millisSinceMuon = millis() - event.time;

          int maxMuonFrothTime = 10000;
          if (millisSinceMuon < event.frothTime) {
            float frothiness = (1 - (float)millisSinceMuon / maxMuonFrothTime);

            if (random(1) < frothiness) {
              float angle = random(0, 2 * PI);
              int x = int(event.x + random(2) * cos(angle));
              int y = int(event.y + random(2) * sin(angle));

              //println(x, y);



              blinkXY(constrain(x, 0, 10), constrain(y, 0, 18), 0, 1, int(minFade + random(maxFade)));
              try {
                Thread.sleep(10);
              } 
              catch (InterruptedException e) {
              }
            }
          } else {
            eventItr.remove();
          }
        }
      }

      try {
        Thread.sleep(10);
      } 
      catch (InterruptedException e) {
      }
    }
  }
}

class MuonTracker implements Runnable {
  private Serial port;

  public MuonTracker(Serial port) {
    this.port = port;
  }

  public void run() {
    while (true) {
      if (port == null) {
        continue;
      }

      //println("Waiting for muon event");
      String event = port.readStringUntil('\n');
      if (event == null || event.length() == 0) {
        try {
          Thread.sleep(50);
        } 
        catch (InterruptedException e) {
        }
        continue;
      }
      event = event.trim();
      if (event.startsWith("Waiting")) {
        // Ignore initialization
        try {
          Thread.sleep(50);
        } 
        catch (InterruptedException e) {
        }
        continue;
      }

      String[] parts = event.split(",");

      int muonX = Integer.parseInt(parts[0]);
      int muonY = Integer.parseInt(parts[1]);

      updateOLED(muonX, muonY);
      println("Muon at (" + muonX + ", " + muonY + "), "  + hour() + ":" + minute());


      muonX = (int)map(Integer.parseInt(parts[0]), 1, 6, 1, 12);
      muonY = (int)map(Integer.parseInt(parts[1]), 1, 6, 1, 18);


      muonX_draw = (int)map(muonX, 1, 12, 0 + drawScale, width - drawScale);
      muonY_draw = (int)map(muonY, 1, 18, 0 + drawScale, height - drawScale);

      draw = true;


      int frothy = (int)(minFrothTime + random(maxFrothTime));
      MuonEvent muonEvent = new MuonEvent(muonX, muonY, frothy);
      frothy = frothy/1000;
      println("overall fade time: " + frothy + " seconds");
      //println("time: " + hour() + ":" + minute());
      synchronized (sensorEvents) {
        sensorEvents.add(muonEvent);
      }
    }
  }
}

public void updateOLED(int xLoc, int yLoc) {
  println("UPDATE OLED");
  String OLED_IP = "192.168.0.146";
  //updatedisplay?X=2&Y=1
  Client OLED = new Client(this, OLED_IP, 80); // Connect to server on port 80
  
  int hour = hour();
  int minute = minute();
  
  //println("Muon at (" + muonX + ", " + muonY + "), "  + hour() + ":" + minute());

  OLED.write("GET /updatedisplay?X=" + xLoc + "&Y=" +yLoc + "&hour=" + hour + "&minute=" + minute + " HTTP/1.0\r\n \r\n"); // Use the HTTP "GET" command to ask for a Web page
}

void drawMuon() {
  fill(0);
  background(255);
  ellipse(muonX_draw, muonY_draw, drawScale, drawScale);
}
