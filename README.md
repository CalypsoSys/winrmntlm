# winrmntlm
Supporting AllowUnencrypted=true for github.com/masterzen/winrm

Sample code below
```
package main

import (
	"context"
	"fmt"
	"os"

	"github.com/CalypsoSys/winrmntlm"
	"github.com/masterzen/winrm"
)

func test_winrmntlm() {
	// winrm set winrm/config/service '@{AllowUnencrypted="true"}'
	// use to fails with ==> func() winrm.Transporter { return &winrm.ClientNTLM{} },
	// now works with ==> winrm.NewEncryption("ntlm")
	//
	// using https/5986
	runExec_winrmntlm("AllowUnencrypted_false_address", 5986, true, "username", "password")

	// winrm set winrm/config/service '@{AllowUnencrypted="true"}'
	// should wortk with both
	//
	// using http/5985
	runExec_winrmntlm("AllowUnencrypted_true_address", 5985, false, "username", "password") // works
}

func runExec_winrmntlm(address string, port int, https bool, userName string, password string) {
	endpoint := winrm.NewEndpoint(address, port, https, true, nil, nil, nil, 0)

	params := winrm.DefaultParameters
	enc, _ := winrmntlm.NewEncryption("ntlm", userName, password, endpoint)
	params.TransportDecorator = func() winrm.Transporter { return enc }

	client, err := winrm.NewClientWithParameters(endpoint, userName, password, params)
	if err != nil {
		fmt.Println(err)
	}

	exitCode, err := client.RunWithContext(context.Background(), "ipconfig /all", os.Stdout, os.Stderr)
	fmt.Printf("%d\n%v\nn", exitCode, err)
	if err != nil {
		_ = exitCode
		fmt.Println(err)
	} else {
		fmt.Println("Command Test Ok")
	}

	wmiQuery := `select * from Win32_ComputerSystem`
	psCommand := fmt.Sprintf(`$FormatEnumerationLimit=-1;  Get-WmiObject -Query "%s" | Out-String -Width 4096`, wmiQuery)
	stdOut, stdErr, exitCode, err := client.RunPSWithContextWithString(context.Background(), psCommand, "")
	fmt.Printf("%d\n%v\n%s\n%s\n", exitCode, err, stdOut, stdErr)
	if err != nil || (len(stdOut) == 0 && len(stdErr) > 0) {
		_ = exitCode
		fmt.Println(err)
	} else {
		fmt.Println("PowerShell Test Ok")
	}
}
