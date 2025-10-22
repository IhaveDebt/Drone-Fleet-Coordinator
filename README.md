// DroneFleetCoordinator.java
// Simulated Drone Fleet Coordinator using TCP sockets
// - Coordinator (server) accepts connections from drone clients
// - Drones periodically send telemetry (id,x,y,z,velocity) and receive commands
//
// Run coordinator: javac DroneFleetCoordinator.java && java DroneFleetCoordinator server
// Run drone client: javac DroneFleetCoordinator.java && java DroneFleetCoordinator drone <id>
import java.io.*;
import java.net.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.time.*;

public class DroneFleetCoordinator {
    static final int PORT = 9999;

    // Start server or client depending on args
    public static void main(String[] args) throws Exception {
        if (args.length == 0 || args[0].equalsIgnoreCase("server")) {
            new Coordinator().start();
        } else if (args[0].equalsIgnoreCase("drone")) {
            String id = args.length > 1 ? args[1] : "drone-" + new Random().nextInt(1000);
            new DroneClient(id).start();
        } else {
            System.out.println("Usage: java DroneFleetCoordinator [server|drone <id>]");
        }
    }

    static class Coordinator {
        private final ExecutorService pool = Executors.newCachedThreadPool();
        private final ConcurrentMap<String, String> latestTelemetry = new ConcurrentHashMap<>();
        private final AtomicInteger connections = new AtomicInteger(0);

        void start() throws IOException {
            ServerSocket ss = new ServerSocket(PORT);
            System.out.println("Coordinator listening on port " + PORT);
            pool.submit(this::monitorLoop);
            while (true) {
                Socket s = ss.accept();
                connections.incrementAndGet();
                pool.submit(() -> handleConnection(s));
            }
        }

        void handleConnection(Socket s) {
            try (BufferedReader r = new BufferedReader(new InputStreamReader(s.getInputStream()));
                 BufferedWriter w = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()))) {
                String line;
                while ((line = r.readLine()) != null) {
                    // simple telemetry format: id|x|y|z|vel
                    if (line.startsWith("TEL|")) {
                        String payload = line.substring(4);
                        String[] parts = payload.split("\\|");
                        String id = parts[0];
                        latestTelemetry.put(id, payload + "|" + Instant.now().toString());
                        // simple command: if z < 1.0, command ascend
                        double z = Double.parseDouble(parts[3]);
                        if (z < 1.0) {
                            w.write("CMD|ASCEND|1.5\n"); w.flush();
                        } else {
                            w.write("CMD|HOLD\n"); w.flush();
                        }
                    } else if (line.startsWith("PING")) {
                        w.write("PONG\n"); w.flush();
                    }
                }
            } catch (IOException e) {
                // connection closed
            } finally {
                connections.decrementAndGet();
            }
        }

        void monitorLoop() {
            try {
                while (true) {
                    Thread.sleep(2000);
                    System.out.println("=== Fleet status ===");
                    System.out.println("Connections: " + connections.get());
                    latestTelemetry.forEach((id, tel) -> System.out.println(" " + id + " -> " + tel));
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    static class DroneClient {
        final String id;
        final Random rnd = new Random();
        DroneClient(String id) { this.id = id; }
        void start() {
            while (true) {
                try (Socket s = new Socket("localhost", PORT);
                     BufferedReader r = new BufferedReader(new InputStreamReader(s.getInputStream()));
                     BufferedWriter w = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()))) {
                    System.out.println("Connected as " + id);
                    for (int t = 0; t < 1000; t++) {
                        double x = rnd.nextDouble() * 100;
                        double y = rnd.nextDouble() * 100;
                        double z = rnd.nextDouble() * 5;
                        double vel = rnd.nextDouble() * 10;
                        String msg = String.format("TEL|%s|%.2f|%.2f|%.2f|%.2f", id, x, y, z, vel);
                        w.write(msg + "\n"); w.flush();
                        // read possible command (with small timeout)
                        s.setSoTimeout(500);
                        try {
                            String cmd = r.readLine();
                            if (cmd != null) {
                                System.out.println("[" + id + "] CMD <- " + cmd);
                                // naive: react to ascend
                                if (cmd.contains("ASCEND")) {
                                    // simulate ascend by increasing z next iteration
                                }
                            }
                        } catch (SocketTimeoutException ste) {
                            // nothing from server
                        }
                        Thread.sleep(1000 + rnd.nextInt(500));
                    }
                } catch (IOException | InterruptedException e) {
                    System.err.println("Connection error, retrying in 2s: " + e.getMessage());
                    try { Thread.sleep(2000); } catch (InterruptedException ex) { break; }
                }
            }
        }
    }
}
