#!/usr/bin/env python3
"""
Simple multithreaded TCP port scanner (educational).
Only scan targets you own or have permission to test.
"""

import socket
import threading
import argparse
from queue import Queue
import time

# Worker thread count
NUM_THREADS = 100

def scan_port(target_ip, port, timeout=1.0):
    """Attempt to connect to target_ip:port. Return True if open."""
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(timeout)
            result = s.connect_ex((target_ip, port))
            return result == 0
    except Exception:
        return False

def worker(target_ip, q, open_ports, lock, timeout):
    while True:
        port = q.get()
        if port is None:
            q.task_done()
            break
        if scan_port(target_ip, port, timeout):
            with lock:
                open_ports.append(port)
                print(f"[+] Port {port} OPEN")
        q.task_done()

def run_scan(target, ports, num_threads=NUM_THREADS, timeout=1.0):
    # Resolve target to IP
    try:
        target_ip = socket.gethostbyname(target)
    except socket.gaierror:
        print(f"[-] Could not resolve {target}")
        return []

    q = Queue()
    open_ports = []
    lock = threading.Lock()

    # Start worker threads
    threads = []
    for _ in range(num_threads):
        t = threading.Thread(target=worker, args=(target_ip, q, open_ports, lock, timeout), daemon=True)
        t.start()
        threads.append(t)

    # Enqueue ports
    for p in ports:
        q.put(p)

    # Block until all tasks are done
    q.join()

    # Stop workers
    for _ in threads:
        q.put(None)
    for t in threads:
        t.join(timeout=0.1)

    return sorted(open_ports)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Simple multithreaded TCP port scanner (educational).")
    parser.add_argument("target", help="Target hostname or IP (only scan if you have permission).")
    parser.add_argument("--start", type=int, default=1, help="Start port (default 1)")
    parser.add_argument("--end", type=int, default=1024, help="End port (default 1024)")
    parser.add_argument("--threads", type=int, default=100, help="Number of threads (default 100)")
    parser.add_argument("--timeout", type=float, default=1.0, help="Socket timeout seconds (default 1.0)")
    parser.add_argument("--out", default="open_ports.txt", help="Output file for open ports")
    args = parser.parse_args()

    ports = range(max(1, args.start), min(65535, args.end) + 1)
    start_time = time.time()
    print(f"Scanning {args.target} ports {args.start}-{args.end} with {args.threads} threads...")
    open_ports = run_scan(args.target, ports, num_threads=args.threads, timeout=args.timeout)
    elapsed = time.time() - start_time

    # Save results
    with open(args.out, "w") as f:
        for p in open_ports:
            f.write(f"{p}\n")

    print(f"\nScan completed in {elapsed:.2f}s. Open ports: {open_ports}")
    print(f"Results saved to {args.out}")
