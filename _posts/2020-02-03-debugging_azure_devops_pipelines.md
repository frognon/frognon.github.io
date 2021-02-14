# Call me Frognon.

Occasionally working with Azure puts me in such a mood that it requires a strong moral principle to prevent me from deliberately context switching into Slack and methodically whining people's hats off. This is my substitute for deleting all resource groups and becoming a fisherman.

Having spent a significant amount of time, never mind how long precisely, trying to upgrade the Terraform Azurerm provider, I managed to come up with a simple way of looking inside the pipeline.

---

## Warnings
As always, don't do this unless you know what you are getting into.

## Why
This enabled me to run certain commands within the pipeline pseudo-interactively, observe their outputs and try other things in response to those outputs. Changing the pipeline definition and re-running the pipeline was far, far slower than this method.

## Requirements
I've used this on the Azure DevOps pipeline, both on Windows and Linux build hosts, but it should be applicable to pretty much any similar environment.

Things needed:

1. A Linux-adjacent (Linux, BSD, WSL, whatever) host, accessible via `ssh`, I will refer to this as `debug-host`
2. Two FIFOs on the `debug-host`
3. Ability to insert arbitrary steps into the build pipeline

I used a small virtual machine running Ubuntu on Azure for this, with the bash shell.

Generate a temporary key for this debugging task. Be sure to not re-use it anywhere and get rid of it once you're done.

```
ssh-keygen -t ed25519 -f temp_key
```

## Debug-host preparation

Append the public key of the newly generated key to the `authorized_keys` on the debug-host.

```
echo "PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
```

Create two FIFOs, in a temporary folder for easy cleanup later.

```
mkdir ~/tmp_debug
mkfifo ~/tmp_debug/to_azure
mkfifo ~/tmp_debug/from_azure
```

Open two sessions to the `debug-host` (or use `tmux` and split a pane).

In the first session, start reading input from the `from_azure` FIFO. Set it up in an infinite loop to continue reading if/when a EOF appears in the FIFO.

```
while true
do
echo "Starting reading at `date`" # informational label when starting
cat < ~/tmp_debug/from_azure
done
```

In the second session, start writing into the `to_azure` FIFO. Again, set it up in an infinite loop to continue writing if/when the receiver goes away/restarts.

```
while true
do
echo "Starting writing at `date`" # informational label when starting
cat > ~/tmp_debug/to_azure
done
```

Keep these two sessions available


## Azure DevOps pipeline preparation

You'll need to make the private key available within the pipeline. To preserve newlines in the base64 encoded key file, base64 encode the key again. On something Unix-ish, you can use the `base64` utility that is usually available. 

```
base64 temp_key | tr -d '\n'
```

On Powershell you can use

```
$curDir = pwd 
[convert]::ToBase64String([IO.File]::ReadAllBytes([IO.Path]::Join($curDir,".\temp_key")))
```

Copy the base64 encoded private key and make it available to the pipeline in the following steps.

Add the following steps into the pipeline at the point where you wish to start executing commands interactively within the running pipeline.

### Steps for a Linux based build host

```
 - bash: echo "XXX" | base64 -d > temp_key
 - bash: chmod 0600 temp_key
 - bash: while [ ! -e stop_this_madness ]; do echo -ne 'start at ' ; date ;  ssh -o 'StrictHostKeyChecking no' -i temp_key azureuser@DEBUG-HOST-ADDRESS 'cat tmp_debug/to_az' | sh - 2>&1 | ssh -o 'StrictHostKeyChecking no' -i temp_key azureuser@DEBUG-HOST-ADDRESS 'cat > tmp_debug/from_az'; done ; rm -f stop_this_madness ; rm -f temp_key
```

### Steps for a Windows based build host

```
- pwsh: write-output "XXX" | Out-File -Encoding ascii -NoNewline -FilePath temp_key.b64
- pwsh: $curDir = pwd ; [System.Text.Encoding]::UTF8.GetString( [convert]::FromBase64String(
[IO.file]::ReadAllText("temp_key.b64"))) | Out-file -Encoding ascii -NoNewline temp_key
- script: icacls key /inheritance:r
- script: icacls key /grant:r "%username%":"R"
- pwsh: |
    do
    {
        write-host "Connecting to remote"
        date
        ssh -o 'StrictHostKeyChecking no' -i temp_key
azureuser@DEBUG-HOST-ADDRESS 'cat tmp_debug/to_az'  | pwsh  -NoLogo -NoProfile
-Command - | ssh -o 'StrictHostKeyChecking no' -i temp_key
azureuser@DEBUG-HOST-ADDRESS 'cat > tmp_debug/from_az'
    }
           While((test-path .\stop_this_madness) -eq 0) ; Remove-Item
./stop_this_madness ; Remove-Item temp_key.b64 ; Remove-Item temp_key
```


Replace `XXX` with the base64 encoded private key, replace `DEBUG-HOST-ADDRESS` with the debug-host address. Replace `azureuser` with whatever username you are using on the debug-host, `azureuser` is appropriate for an Azure host with default parameters.

These steps write and decode the base64 encoded key to a file called `temp_key`, change permissions on the key so that `ssh` will accept it (make it only readable by the owner). Once the key is decoded into the `temp_key` file, `ssh` is used to connect to the debug-host, where input is read from the `to_azure` FIFO. This input is piped as a command into a local shell (bash on Linux, powershell on Windows), running within the pipeline. Output from the shell is sent over `ssh` into the `from_azure` FIFO on the debug-host.

You can type commands into the debug-host session which is writing into the `to_azure` FIFO and observe output from the command, executed within the pipeline, in the session reading from the `from_azure` FIFO. Note that you need to hit return twice after a command to execute commands on a  Windows build host.

The pseudo-interactive shell session within the build pipeline will execute until the pipeline times out, or until a file named `stop_this_madness` is found within the pipeline, when the loop running `ssh` within the pipeline cycles. To stop a session running within a Linux build host you could enter the commands

```
touch stop_this_madness
exit
```

into the session writing into the `to_azure` FIFO. A session running on within a Windows build host you can use

```
Out-File stop_this_madness
[Environment]::Exit(1)
```


## Cleanup
Remember to remove the public `temp_key` from the `debug-host`'s `authorized_keys`, if you plan on keeping the host around. Remove the steps from the build pipeline. Shut down the sessions reading/writing to the FIFOs.

## Notes
There probably exist cleaner methods of connecting a shell over ssh, executing the shell within the pipeline and connecting it's standard input/output to something on the ssh host side. I wasn't able to immediately think of how to do that, so I did this instead. I'd be moderately interested in learning how to do that, you can tweet me @frognon1.
