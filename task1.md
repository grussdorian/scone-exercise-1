## Task 1

### Problem

We want to create a confidential program `P1` that

1. takes all its arguments, writes each arguments as a line in file `/volumes/v1/file`

2. `P1` should run under a policy `N`

3. It asks for an OTP to be able to run

4. `P1` should take the host arguments (i.e., arguments provided by user)

5. Host arguments: specify with `@@1` (first host argument) `@@2` (second host argument), In the policy, command: `print-arg-env arg1 @@1 @@2`
   Mix with arguments from policy: e.g., `arg1` is provided by policy

6. Ensure that file `/volumes/v1` is transparently encrypted

7. Define the volume in a separate policy `V`

8. `V` exports the volume to policy `P1`. Export the volume to namespace `<N>`

### Solution

**All the policies are under the folder `policies`**
**All the logs are under the folder `logs`**
**The encrypted files have been copied under the folder called `encrypted_files`**

First we define some prerequisites and write them in our `.bashrc` file so we don't have to write verbose commands again and again. In the `.bashrc` file:

```bash
..
..

alias scone="docker run $MOUNT_SGXDEVICE  --network=host -it -v `pwd`:/work registry.scontain.com/sconecuratedimages/crosscompilers bash"

function determine_sgx_device {
    export SGXDEVICE="/dev/sgx_enclave"
    export MOUNT_SGXDEVICE="--device=/dev/sgx_enclave"
    if [[ ! -e "$SGXDEVICE" ]] ; then
        export SGXDEVICE="/dev/sgx"
        export MOUNT_SGXDEVICE="--device=/dev/sgx"
        if [[ ! -e "$SGXDEVICE" ]] ; then
            export SGXDEVICE="/dev/isgx"
            export MOUNT_SGXDEVICE="--device=/dev/isgx"
            if [[ ! -c "$SGXDEVICE" ]] ; then
                echo "Warning: No SGX device found! Will run in SIM mode." > /dev/stderr
                export MOUNT_SGXDEVICE=""
                export SGXDEVICE=""
            fi
        fi
    fi
}

determine_sgx_device
```

Then we run the docker container with the correct SGX device

```bash
scone
```

The source file of the C program `P1` used is given here. Since the docker container we have is very minimal, we use the `cat` command to write and read files due to the unavailability of text editors like nano and vim. The working directory is called `/work`

```bash
cd /work
cat > P1.c << EOF
# Then the rest of the c program ...
```

Now we have to compile the program (optimisation level 3 and with the debugging information attached)

```bash
scone-gcc P1.c -g -O3 -o P1
```

Now running the program

```bash
mkdir -p /volumes/v1/
./P1 hello world
```

Running of this program has created the file `file.txt` with unencrypted contents. Output in file `log1`

The output is unencrypted. We want to encrypt it. For that we need to create a scone policy with a session and attest it from CAS.

But first we want to create a namespace called `exercise_1` and then we would create two different policies one for `P1` and another for our volume `V` which run under `exercise_1` namespace.

File `p1_namespace.yaml` defines our namespace. Then we create two different files `policy_for_V.yaml` and `policy_for_p1.yaml` as the names suggest.

To create a policy `N`, we do the following:

```bash
export SESSION_P1=P1-$RANDOM-$RANDOM
export SESSION_V=V-$RANDOM-$RANDOM
export MRENCLAVE=`SCONE_HASH=1 ./P1 Assignment Hardik Ghoshal 5184856`
```

We are basically running `P1` to find its hash called `MRENCLAVE` that we need to use in our policy to ensure integrity of the code while running.

The content of the file `/volumes/v1/file.txt` is shown in `output1.log` file. The contents are in clear text.

The $RANDOM will generate a random number every time we want to create a session. This means that for every session we create, we get a different name. This is really convenient as to avoid duplicate session names without much hassle.

Now we have to specify where CAS and LAS are running.

```bash
export SCONE_CAS_ADDR=141.76.44.93
export SCONE_LAS_ADDR=141.76.50.190
```

And we have to define our OTPSECRET variable which stores the base 32 encoded value of a string which we arbitrarily chose. Our string `iamtestingsconehello` and our base 32 encoded version of the string is `NFQW25DFON2GS3THONRW63TFNBSWY3DP`

one can easily encode their string into a valid base32 encoding from this [website](https://emn178.github.io/online-tools/base32_encode.html)

Before running the code we want to create sessions that we previously defined in our policy files.

```bash
export PREDECESSOR_namespace=$(scone session create p1_namespace.yaml)
export PREDECESSOR_V=$(scone session create policy_for_V.yaml)
export PREDECESSOR_P1=$(scone session create policy_for_P1.yaml)


```

The predecessor variable is used when we want to chain policies further (for example if we want to decrypt the file we would need the predecessor value)

Finally before running the code, we derive an OTP from [this website](https://totp.info/) where we have to put our `OTPSECRET`

Then

```bash
export OTP=123456 # correct otp from the website
```

Output of running the list directory command before running the code is shown in `output2.log` file.

And then running the program

```bash
SCONE_CONFIG_ID=exercise_1/$SESSION_P1/P1@$OTP ./P1 Hardik Ghoshal
```

Finally output of running the list directory command after running the code is given in `output3.log`
