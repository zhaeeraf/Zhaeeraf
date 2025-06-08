## Hi there 👋

<!--
**zhaeeraf/Zhaeeraf** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
#!/usr/bin/env python3
"""
Advanced Visitor Tracker - Enhanced Flask Application
Demonstrates socket programming concepts with visitor logging and analytics
"""

from flask import Flask, request, jsonify, render_template_string
from datetime import datetime, timedelta
import json
import os
import sqlite3
import threading
from collections import defaultdict
import socket
import geoip2.database
import geoip2.errors

class VisitorTracker:
    def __init__(self, db_path='visitors.db'):
        self.db_path = db_path
        self.init_database()
        self.visitor_stats = defaultdict(int)
        self.lock = threading.Lock()
        
    def init_database(self):
        """Initialize SQLite database for visitor tracking"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS visitors (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp TEXT NOT NULL,
                ip_address TEXT NOT NULL,
                user_agent TEXT,
                path TEXT NOT NULL,
                method TEXT NOT NULL,
                country TEXT,
                city TEXT,
                hostname TEXT,
                headers TEXT
            )
        ''')
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS visit_stats (
                date TEXT PRIMARY KEY,
                unique_visitors INTEGER,
                total_requests INTEGER,
                top_paths TEXT,
                top_countries TEXT
            )
        ''')
        
        conn.commit()
        conn.close()
    
    def get_hostname(self, ip):
        """Get hostname from IP address"""
        try:
            return socket.gethostbyaddr(ip)[0]
        except:
            return "Unknown"
    
    def get_location(self, ip):
        """Get location from IP (placeholder - would need GeoIP database)"""
        # In a real implementation, you'd use a GeoIP database
        # For demonstration, we'll return placeholder data
        if ip.startswith('127.') or ip.startswith('192.168.') or ip.startswith('10.'):
            return "Local", "Local Network"
        return "Unknown", "Unknown"
    
    def log_visit(self, ip, user_agent, path, method, headers):
        """Log a visitor with detailed information"""
        timestamp = datetime.now().isoformat()
        hostname = self.get_hostname(ip)
        country, city = self.get_location(ip)
        
        with self.lock:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            cursor.execute('''
                INSERT INTO visitors 
                (timestamp, ip_address, user_agent, path, method, country, city, hostname, headers)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (timestamp, ip, user_agent, path, method, country, city, hostname, json.dumps(dict(headers))))
            
            conn.commit()
            conn.close()
            
            # Update in-memory stats
            self.visitor_stats[ip] += 1
    
    def get_recent_visitors(self, limit=50):
        """Get recent visitors"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT timestamp, ip_address, user_agent, path, method, country, city, hostname
            FROM visitors 
            ORDER BY timestamp DESC 
            LIMIT ?
        ''', (limit,))
        
        visitors = cursor.fetchall()
        conn.close()
        
        return [
            {
                'timestamp': v[0],
                'ip': v[1],
                'user_agent': v[2],
                'path': v[3],
                'method': v[4],
                'country': v[5],
                'city': v[6],
                'hostname': v[7]
            }
            for v in visitors
        ]
    
    def get_stats(self):
        """Get visitor statistics"""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        # Total visitors
        cursor.execute('SELECT COUNT(*) FROM visitors')
        total_visits = cursor.fetchone()[0]
        
        # Unique IPs
        cursor.execute('SELECT COUNT(DISTINCT ip_address) FROM visitors')
        unique_visitors = cursor.fetchone()[0]
        
        # Recent activity (last 24 hours)
        yesterday = (datetime.now() - timedelta(days=1)).isoformat()
        cursor.execute('SELECT COUNT(*) FROM visitors WHERE timestamp > ?', (yesterday,))
        recent_activity = cursor.fetchone()[0]
        
        # Top paths
        cursor.execute('''
            SELECT path, COUNT(*) as count 
            FROM visitors 
            GROUP BY path 
            ORDER BY count DESC 
            LIMIT 10
        ''')
        top_paths = dict(cursor.fetchall())
        
        # Top countries
        cursor.execute('''
            SELECT country, COUNT(*) as count 
            FROM visitors 
            WHERE country != 'Unknown'
            GROUP BY country 
            ORDER BY count DESC 
            LIMIT 10
        ''')
        top_countries = dict(cursor.fetchall())
        
        conn.close()
        
        return {
            'total_visits': total_visits,
            'unique_visitors': unique_visitors,
            'recent_activity': recent_activity,
            'top_paths': top_paths,
            'top_countries': top_countries
        }

app = Flask(__name__)
tracker = VisitorTracker()

# HTML template for the dashboard
DASHBOARD_HTML = '''
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Visitor Tracker Dashboard</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/bootstrap.min.css" rel="stylesheet">
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <div class="container-fluid">
        <header class="row bg-dark text-white py-3 mb-4">
            <div class="col">
                <h1><i class="fas fa-eye me-2"></i>Visitor Tracker Dashboard</h1>
                <p class="mb-0 opacity-75">Real-time visitor analytics and socket programming insights</p>
            </div>
        </header>

        <div class="row mb-4">
            <div class="col-md-3">
                <div class="card bg-primary text-white">
                    <div class="card-body">
                        <div class="d-flex justify-content-between">
                            <div>
                                <h4 id="totalVisits">{{ stats.total_visits }}</h4>
                                <p>Total Visits</p>
                            </div>
                            <i class="fas fa-users fa-2x"></i>
                        </div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="card bg-success text-white">
                    <div class="card-body">
                        <div class="d-flex justify-content-between">
                            <div>
                                <h4 id="uniqueVisitors">{{ stats.unique_visitors }}</h4>
                                <p>Unique Visitors</p>
                            </div>
                            <i class="fas fa-user-check fa-2x"></i>
                        </div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="card bg-info text-white">
                    <div class="card-body">
                        <div class="d-flex justify-content-between">
                            <div>
                                <h4 id="recentActivity">{{ stats.recent_activity }}</h4>
                                <p>Last 24 Hours</p>
                            </div>
                            <i class="fas fa-clock fa-2x"></i>
                        </div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="card bg-warning text-white">
                    <div class="card-body">
                        <div class="d-flex justify-content-between">
                            <div>
                                <h4 id="liveConnections">Live</h4>
                                <p>Socket Programming</p>
                            </div>
                            <i class="fas fa-network-wired fa-2x"></i>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="row">
            <div class="col-lg-8">
                <div class="card">
                    <div class="card-header">
                        <h5><i class="fas fa-list me-2"></i>Recent Visitors</h5>
                    </div>
                    <div class="card-body">
                        <div class="table-responsive">
                            <table class="table table-striped">
                                <thead>
                                    <tr>
                                        <th>Time</th>
                                        <th>IP Address</th>
                                        <th>Location</th>
                                        <th>Path</th>
                                        <th>User Agent</th>
                                    </tr>
                                </thead>
                                <tbody>
                                    {% for visitor in visitors %}
                                    <tr>
                                        <td><small>{{ visitor.timestamp.split('T')[1].split('.')[0] }}</small></td>
                                        <td><code>{{ visitor.ip }}</code></td>
                                        <td><small>{{ visitor.country }}, {{ visitor.city }}</small></td>
                                        <td><span class="badge bg-primary">{{ visitor.path }}</span></td>
                                        <td><small class="text-truncate" style="max-width: 200px;">{{ visitor.user_agent }}</small></td>
                                    </tr>
                                    {% endfor %}
                                </tbody>
                            </table>
                        </div>
                    </div>
                </div>
            </div>

            <div class="col-lg-4">
                <div class="card mb-3">
                    <div class="card-header">
                        <h6><i class="fas fa-chart-bar me-1"></i>Top Paths</h6>
                    </div>
                    <div class="card-body">
                        {% for path, count in stats.top_paths.items() %}
                        <div class="d-flex justify-content-between mb-2">
                            <span><code>{{ path }}</code></span>
                            <span class="badge bg-secondary">{{ count }}</span>
                        </div>
                        {% endfor %}
                    </div>
                </div>

                <div class="card">
                    <div class="card-header">
                        <h6><i class="fas fa-globe me-1"></i>Socket Programming Insights</h6>
                    </div>
                    <div class="card-body">
                        <div class="alert alert-info">
                            <h6><i class="fas fa-info-circle me-1"></i>Network Analysis</h6>
                            <ul class="mb-0 small">
                                <li>Each visitor creates a TCP socket connection</li>
                                <li>HTTP headers reveal client capabilities</li>
                                <li>IP addresses show geographic distribution</li>
                                <li>Hostnames provide reverse DNS insights</li>
                            </ul>
                        </div>
                        
                        <div class="alert alert-success">
                            <h6><i class="fas fa-code me-1"></i>Real-time Tracking</h6>
                            <p class="mb-0 small">This tracker demonstrates socket-level monitoring of HTTP connections and request patterns.</p>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // Auto-refresh every 30 seconds
        setTimeout(() => {
            location.reload();
        }, 30000);
    </script>
</body>
</html>
'''

@app.before_request
def log_visitor():
    """Log every incoming request"""
    ip = request.remote_addr
    user_agent = request.headers.get('User-Agent', 'Unknown')
    path = request.path
    method = request.method
    headers = request.headers
    
    tracker.log_visit(ip, user_agent, path, method, headers)

@app.route('/')
def index():
    """Main welcome page"""
    ip = request.remote_addr
    user_agent = request.headers.get('User-Agent', 'Unknown')
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    # Also write to file for compatibility
    with open("ips.txt", "a") as f:
        f.write(f"Time: {timestamp} | IP: {ip} | UA: {user_agent}\n")
    
    return f'''
    <h1>🌐 Welcome to the Enhanced Visitor Tracker!</h1>
    <p>Your visit has been logged with advanced socket programming techniques.</p>
    <div style="background: #f8f9fa; padding: 20px; border-radius: 8px; margin: 20px 0;">
        <h3>📊 Your Connection Details:</h3>
        <p><strong>IP Address:</strong> <code>{ip}</code></p>
        <p><strong>User Agent:</strong> <code>{user_agent}</code></p>
        <p><strong>Timestamp:</strong> {timestamp}</p>
        <p><strong>Hostname:</strong> {tracker.get_hostname(ip)}</p>
    </div>
    <p><a href="/dashboard" style="background: #007bff; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px;">📈 View Dashboard</a></p>
    <p><a href="/api/visitors" style="background: #28a745; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px;">📋 JSON API</a></p>
    '''

@app.route('/dashboard')
def dashboard():
    """Visitor analytics dashboard"""
    stats = tracker.get_stats()
    visitors = tracker.get_recent_visitors(20)
    
    return render_template_string(DASHBOARD_HTML, stats=stats, visitors=visitors)

@app.route('/api/visitors')
def api_visitors():
    """JSON API endpoint for visitor data"""
    visitors = tracker.get_recent_visitors()
    return jsonify({
        'success': True,
        'visitors': visitors,
        'count': len(visitors)
    })

@app.route('/api/stats')
def api_stats():
    """JSON API endpoint for statistics"""
    stats = tracker.get_stats()
    return jsonify({
        'success': True,
        'stats': stats
    })

@app.route('/test')
def test_endpoint():
    """Test endpoint for generating traffic"""
    return jsonify({
        'message': 'Test endpoint hit successfully',
        'ip': request.remote_addr,
        'timestamp': datetime.now().isoformat(),
        'socket_info': {
            'remote_addr': request.remote_addr,
            'method': request.method,
            'path': request.path,
            'headers_count': len(request.headers)
        }
    })

@app.route('/socket-info')
def socket_info():
    """Detailed socket and network information"""
    ip = request.remote_addr
    hostname = tracker.get_hostname(ip)
    
    socket_details = {
        'client_ip': ip,
        'client_hostname': hostname,
        'server_host': request.host,
        'method': request.method,
        'protocol': request.environ.get('SERVER_PROTOCOL'),
        'user_agent': request.headers.get('User-Agent'),
        'accept_encoding': request.headers.get('Accept-Encoding'),
        'connection': request.headers.get('Connection'),
        'headers': dict(request.headers)
    }
    
    return f'''
    <h1>🔌 Socket Programming Information</h1>
    <div style="background: #f8f9fa; padding: 20px; border-radius: 8px; font-family: monospace;">
        <h3>TCP Connection Details:</h3>
        <pre>{json.dumps(socket_details, indent=2)}</pre>
    </div>
    <p><a href="/" style="background: #6c757d; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px;">← Back to Home</a></p>
    '''

if __name__ == "__main__":
    print("🚀 Starting Enhanced Visitor Tracker...")
    print("🌐 Main page: http://localhost:8000")
    print("📊 Dashboard: http://localhost:8000/dashboard")
    print("📋 API: http://localhost:8000/api/visitors")
    print("🔌 Socket Info: http://localhost:8000/socket-info")
    print("=" * 50)
    
    app.run(host='0.0.0.0', port=8000, debug=True)
