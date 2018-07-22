## Tesseract -  A Container that can execute smart contract

이번 포스팅에서 설명할 Tesseact는([마블 시네마틱 유니버스](https://namu.wiki/w/%EB%A7%88%EB%B8%94%20%EC%8B%9C%EB%84%A4%EB%A7%88%ED%8B%B1%20%EC%9C%A0%EB%8B%88%EB%B2%84%EC%8A%A4)에 등장하는 공간을 조종하는 스페이스 스톤)는 독립된 Docker container에 SmartContract을 실행시키는 라이브러리 이다.

![스크린샷 2018-03-11 오후 8.35.42](https://www.dropbox.com/s/hokh6u7hqkxgix9/tesseract.png?dl=1)

Tesseract는 SmartContract를 실행하기 위한 맞춤형 환경 세팅을 제공하며, 블록체인 엔진과의 영향을 최소화 하기 위해 독립된 컨테이너를 통해 SmartContract을 실행한다.



## Architecture

### Logical View

![스크린샷 2018-03-11 오후 8.35.42](https://www.dropbox.com/s/xcgw514arf94dpu/logical-object.png?dl=1)



- Tesseract

  다수의 CellCode를 관리하며 Grpc를 통해 CellCode에게 ICode의 Invoke와 Query요청을 보낸다.

- CellCode

  CellCode는 Docker Container 내부에서 돌아가는 독립적인 프로세스로 ICode를 컨트롤한다. CellCode는 grpc를 통해 Tesseract와 통신하며 Tesseract로 부터 Invoke와 Query요청을 받으면 해당하는 요청을 ICode로 실행시키고 결과값을 리턴한다.

- ICode

  ICode는 It-chain위에 돌아가는 SmartContract이다.



![스크린샷 2018-03-11 오후 8.35.42](https://www.dropbox.com/s/z2etkjxcb5xkenk/logical-runicode-sequenceDiagram.png?dl=1)

1. Tesseract가 Grpc를 통해 CellCode에게 실행 요청을 전송한다.
2. Grpc로 받은 요청에 따라 Invoke혹은 Query요청을 ICode로 실행한다.
3. ICode는 Invoke 혹은 Query요청을 수행하고 결과를 리턴한다.
4. ICode로 부터 받은 결과를 grpc를 통해 결과를 리턴한다.



### Deployment View

![deployment](https://www.dropbox.com/s/pajbppcxm3wwwmn/deployment.png?dl=1)

Tesseract를 Peer에 존재하며 ICode의 Deploy시에 CellCode와 ICode는 DockContainer안으로 들어가 DockerContainer내부에서 실행된다.



## CellCode

```Go
package main

import (
	"os"
	"plugin"
)

type ICode interface {
	Query()
	Invoke()
}

func main() {
	if len(os.Args) != 2 {
		os.Exit(1)
	}

	iCodePath := os.Args[1]

	plug, err := plugin.Open(iCodePath)
	if err != nil {
		os.Exit(1)
	}

	iCodePlugin, err := plug.Lookup("ICodeInstance")
	if err != nil {
		os.Exit(1)
	}

	iCode := iCodePlugin.(ICode)

	// ToDo : Socket Connection
	// ToDo : Setting Cell

	if(true) {
		iCode.Query()
	} else {
		iCode.Invoke()
	}
```

위 코드는 CellCode의 main함수이다. CellCode는 Docker Container 내부에서 돌아가는 독립적인 프로세스로 ICode가 Deploy시에 CellCode와 ICode가 모두 Docker Container로 Copy되며 setup.sh에 의해 실행된다. Go의 plugin라이브러리를 통해 ICode파일을 찾고 파일에 존재하는 ICode객체를 instance화 한후에 컨트롤 한다. 
CellCode가 실행되면 grpc stream을 통해 Peer와 통신하며 Peer로 부터 ICode실행 요청이 jsonrpc형식으로 오게되면 ICode를 실행 시킨후 결과를 반환한다. (아직 grpc stream이 완전히 구현되지 못했다.) 



## ICode

```Go
package main

import (
	"os/exec"
)

type ICode struct {

}

func (ic *ICode) Query() {
	cmd := exec.Command("touch", "/cellcode/query")
	cmd.Run()
}

func (ic *ICode) Invoke() {
}

var ICodeInstance ICode
```

ICode는  It-chain위에 돌아가는 SmartContract으로 필수 함수인 Query와 Invoke를 가지고 있다. Invoke는 DB의 insert, update, delete요청이며 Query는 조회요청이다. 위에서 CellCode는 ICodeInstance를 찾기 때문에 꼭 var ICodeInstance를 선언해주어야 한다. 아직 완전히 구현되지는 않았지만 ICode에서 필요한 모든 외부 리소스(DB interface)등은 CellCode가 Cell이라는 객체를 통해 ICode에게 넘겨준다. ICode는 넘겨 받은 Cell에서 제공하는 interface를 통해 외부 리소스를 컨트롤한다. 



### Docker

```go
package docker

import (
	"context"
	"io"
	"os"

	"docker.io/go-docker"
	"docker.io/go-docker/api/types"
	"docker.io/go-docker/api/types/container"

	"path/filepath"

	"github.com/it-chain/tesseract"
)

const (
	DefaultImageName = "golang"
	DefaultImageTag  = "1.9"
)

func CreateContainerWithCellCode(dockerImage DockerImage, iCodeInfo tesseract.ICodeInfo, tesseractPath string, shPath string) (container.ContainerCreateCreatedBody, error) {

	res := container.ContainerCreateCreatedBody{}
	image := dockerImage.Name + ":" + dockerImage.Tag

	exist, err := HasImage(image)

	if err != nil {
		return res, err
	}

	if !exist {
		if err := PullImage(image); err != nil {
			return res, err
		}
	}

	ctx := context.Background()
	cli, err := docker.NewEnvClient()
	if err != nil {
		return res, err
	}

	res, err = cli.ContainerCreate(ctx, &container.Config{
		Image: dockerImage.Name + ":" + dockerImage.Tag,
		Cmd: []string{
			"sh",
			"/sh/" + filepath.Base(shPath),
		},
		Tty:          true,
		AttachStdout: true,
		AttachStderr: true,
	}, &container.HostConfig{
		CapAdd: []string{"SYS_ADMIN"},
		Binds:  []string{tesseractPath + ":/tesseract", iCodeInfo.Directory + ":/icode", filepath.Dir(shPath) + ":/sh"},
	}, nil, "")

	if err != nil {
		return res, err
	}

	return res, nil
}

func StartContainer(containerBody container.ContainerCreateCreatedBody) error {
	ctx := context.Background()
	cli, err := docker.NewEnvClient()

	err = cli.ContainerStart(ctx, containerBody.ID, types.ContainerStartOptions{})
	if err != nil {
		// An error occurred while starting the container!
		return err
	}

	return nil
}

func PullImage(imageName string) error {

	ctx := context.Background()
	cli, err := docker.NewEnvClient()

	if err != nil {
		return err
	}

	resp, err := cli.ImagePull(ctx, imageName, types.ImagePullOptions{})
	defer resp.Close()

	io.Copy(os.Stdout, resp)

	if err != nil {
		return err
	}

	return nil
}

func HasImage(name string) (bool, error) {
	ctx := context.Background()
	cli, err := docker.NewEnvClient()

	imageList, err := cli.ImageList(ctx, types.ImageListOptions{})

	if err != nil {
		return false, err
	}

	for _, image := range imageList {
		if name == image.RepoTags[0] {
			return true, nil
		}
	}

	return false, nil
}
```

Docker는 Docker.io라이브러리를 활용해서 ICode를  실행시키기 위한 Docker Image 다운로드, Docker Container 생성 및 실행을 담당한다. ICode Deploy요청이 오면 해당하는 ICode에서 요청하는 Docker Image를 세팅한 후에 Docker Container를 실행 시킨다. 



이번 포스팅에서는 it-chain-Engine에서 사용할 Tesseract 라이브러리에 대해서 알아보았습니다. 많은 분들이 SmartContract가 어떻게 실행되는지에 대하여 궁금하신 분들이 많은것 같아서 이렇게 포스팅 합니다. 모든 SmartContract가 위와 같은 방식으로 동작하는 것은 아니지만 hyperledger fabric에서는 위와 비슷한 형태로 SmartContract를 실행하고 있습니다. 구현이 되는대로 추가로 문서를 업데이트 하겠습니다.



[Github주소]: https://github.com/it-chain/tesseract	"Tesseract"

