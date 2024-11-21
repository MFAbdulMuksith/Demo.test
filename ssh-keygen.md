The documentation you've provided is mostly correct, but there are some minor clarifications and potential improvements to ensure that everything works smoothly. Here's the revised version with comments:

---

### **On the Local Machine:**
1. **Generate SSH Key Pair:**
   ```bash
   ssh-keygen
   ```
   - This creates a private key (`id_rsa`) and a public key (`id_rsa.pub`) in `~/.ssh/`.

---

### **On the Server:**
2. **Add a New User:**
   ```bash
   sudo adduser username
   ```
   - Replace `username` with the desired username.
   - This command creates the new user and its home directory.

3. **Grant the User Sudo Privileges:**
   ```bash
   sudo usermod -aG sudo username
   ```
   - This adds the user to the `sudo` group.

4. **Switch to the New User:**
   ```bash
   sudo su - username
   ```
   - The `-` ensures that the new user's environment is loaded.

5. **Set Up SSH Access:**
   ```bash
   mkdir .ssh/
   chmod 700 .ssh/
   ```
   - Creates the `.ssh` directory with secure permissions.

6. **Add Public Key to `authorized_keys`:**
   ```bash
   sudo vim .ssh/authorized_keys
   ```
   - Paste the contents of your local `id_rsa.pub` file here.
   - Save and exit (using `:wq` in `vim`).

7. **Set Permissions:**
   ```bash
   chmod 600 .ssh/authorized_keys
   ```
   - Ensures the file is readable only by the user.

---

### **Back on the Local Machine:**
8. **Convert Private Key for Compatibility:**
   ```bash
   cp id_rsa id_rsa.pem
   ssh-keygen -p -N "" -m pem -f id_rsa.pem
   ```
   - The `-p` option updates the passphrase. Setting `-N ""` removes the passphrase.
   - The `-m pem` option converts the private key to PEM format, useful for older tools or specific systems.

9. **Connect to the Server:**
   ```bash
   ssh -i id_rsa.pem username@server-ip
   ```
   - Replace `username` and `server-ip` with the appropriate values.

---

### **Additional Notes:**
- Ensure the SSH server (`sshd`) is installed and running on the server.
- Check that the file `/etc/ssh/sshd_config` allows key-based authentication:
  ```bash
  sudo vim /etc/ssh/sshd_config
  ```
  Look for:
  ```
  PubkeyAuthentication yes
  ```
  and reload the SSH service if changes are made:
  ```bash
  sudo systemctl reload ssh
  ```

- If converting the private key to PEM format is unnecessary for your use case, steps 8 and 9 can be simplified:
  ```bash
  ssh -i id_rsa username@server-ip
  ```

This revised guide should work as intended.
