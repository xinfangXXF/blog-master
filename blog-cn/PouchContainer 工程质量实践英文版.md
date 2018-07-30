# 0. 前言

随着 [PouchContainer](https://github.com/alibaba/pouch) 功能不断地迭代和完善，项目也逐渐庞大起来，这吸引了不少外部开发者来参与项目的开发。由于每位贡献者编码习惯都不尽相同，代码审阅者的责任不仅仅是关注逻辑正确性和性能问题，还应该关注代码风格，因为统一的代码规范是保证项目代码可维护的前提。除了统一项目代码风格之外，测试用例的覆盖率和稳定性也是项目关注的重点。简单设想下，在缺少回归测试用例的项目，如何保证每次代码更新都不会影响到现有功能？

本文会分享 PouchContainer 在代码风格规范和 golang 单元测试用例方面的实践。

# 1. 统一的编码风格规范

PouchContainer 是由 golang 语言构建的项目，项目里会使用 shell script 来完成一些自动化操作，比如编译和打包操作。除了 golang 和 shell script 以外，PouchContainer 还包含了大量 Markdown 风格的文档，它是使用者认识和了解 PouchContainer 的入口，它的规范排版和正确拼写也是项目的关注对象。接下来的内容将会介绍 PouchContainer 在编码风格规范上使用的工具和使用场景。

## 1.1 Golinter - 统一代码格式

golang 的语法设计简单，加上社区一开始都有完备的 [CodeReview](https://github.com/golang/go/wiki/CodeReviewComments) 指导，让绝大部分的 golang 项目都有相同的代码风格，很少陷入到无谓的 __宗教__ 之争。在社区的基础上，PouchContainer 还定义了一些特定的规则来约定开发者，目的是为了保证代码的可读性，具体内容可阅读[这里](https://github.com/alibaba/pouch/blob/master/docs/contributions/code_styles.md#additional-style-rules)。

但光靠书面协议去做规范，这是很难保证项目代码风格保持一致。因此 golang 和其他语言一样，其官方提供了基础的工具链，比如 [golint](https://github.com/golang/lint),  [gofmt](https://golang.org/cmd/gofmt)，[goimports](https://github.com/golang/tools/blob/master/cmd/goimports/doc.go) 以及 [go vet](https://golang.org/cmd/vet) 等等，这些工具可在编译前检查和统一代码风格，为代码审阅等后续流程提供了自动化的可能。目前 PouchContainer 在 __<span data-type="color" style="color:#F5222D">每一次</span>__ 开发者提的 Pull Request 都会在 CircleCI 运行上述的代码检查工具。如果检查工具显示异常，代码审阅者有权 __<span data-type="color" style="color:#F5222D">拒绝</span>__ 审阅，甚至可以拒绝合并代码。

除了官方提供的工具外，我们还可以在开源社区中选择第三方的代码检查工具，比如 [errcheck](https://github.com/kisielk/errcheck) 检查开发者是否都处理了函数返回的 error 。但是这些工具并没有统一的输出格式，这很难完成不同工具输出结果的整合。好在开源社区有人实现了这一层统一的接口，即 [gometalinter](https://github.com/alecthomas/gometalinter)，它可以整合各种代码检查工具，推荐采用的组合是：

* [golint](https://github.com/golang/lint) - Google's (mostly stylistic) linter.
* [gofmt -s](https://golang.org/cmd/gofmt/) - Checks if the code is properly formatted and could not be further simplified.
* [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports) - Checks missing or unreferenced package imports.
* [go vet](https://golang.org/cmd/vet/) - Reports potential errors that otherwise compile.
* [varcheck](https://github.com/opennota/check) - Find unused global variables and constants.
* [structcheck](https://github.com/opennota/check) - Find unused struct fields
* [errcheck](https://github.com/kisielk/errcheck) - Check that error return values are used.
* [misspell](https://github.com/client9/misspell) - Finds commonly misspelled English words.

每个项目都可以根据自己的需求来订制 gometalinter 套餐。

## 1.2 Shellcheck - 减少 shell script 潜在问题

shell script 虽然功能强大，但是它依然需要语法检查来避免一些潜在的、不可预判的错误。比如定义了未使用的变量，虽然不影响脚本的使用，但是它的存在会成为项目维护者的负担。

```powershell
#!/usr/bin/env bash

pouch_version=0.5.x

dosomething() {
  echo "do something"
}

dosomething
```

PouchContainer 会使用 [shellcheck](https://github.com/koalaman/shellcheck) 来检查目前项目里的 shell script。就以上述代码为例，shellcheck 检测会获得未使用变量的警告。该工具可以在代码审阅阶段发现 shell script 潜在的问题，减少运行时出错的概率。

```plain
In test.sh line 3:
pouch_version=0.5.x
^-- SC2034: pouch_version appears unused. Verify it or export it.
```

PouchContainer 当前的持续集成任务会扫描项目里 `.sh` 脚本，并逐一使用 shellcheck 来检查，详情请查看[这里](https://github.com/alibaba/pouch/blob/master/.circleci/config.yml#L21-L24)。

> NOTE: 当 shellcheck 检查太过于严格了，项目里可以通过加注释的方式来避开检查，或者是项目里统一关闭某项检查。具体的检查规则可查看[这里](https://github.com/koalaman/shellcheck/wiki)。

## 1.3 Markdownlint - 统一文档格式编排

PouchContainer 作为开源项目，它的文档同代码一样重要，因为文档是让用户了解 PouchContainer 的最佳方式。文档采用 markdown 的方式来编写，它的编排格式和拼写错误都是项目重点照顾对象。

同代码一样，光有文本约定还是会出现漏判，所以 PouchContainer 采用 [markdownlint](https://github.com/markdownlint/markdownlint) 和 [misspell](https://github.com/client9/misspell) 来检查文档格式和拼写错误，这些检查的地位同 `golint` 一样，会在每次 Pull Request 都会在 CircleCI 中运行，一旦出现异常，代码审阅者有权 __<span data-type="color" style="color:#F5222D">拒绝</span>__ 审阅或者合并代码。

PouchContainer 当前的持续集成任务会检查项目里的 markdown 文档编排格式，同时还检查了所有文件里的拼写，具体配置可查看[这里](https://github.com/alibaba/pouch/blob/master/.circleci/config.yml#L13-L20)。

> NOTE: 当 markdownlint 要求太过于严格时，项目里可以关闭相应的检查。具体的检查项目可查看[这里](https://github.com/markdownlint/markdownlint/blob/master/docs/RULES.md)。

## 1.4 小结

上述内容都属于风格纪律问题，PouchContainer 将编码规范检测自动化，集成到每一次的代码审阅中，帮助审阅者发现潜在的问题。

# 2. How to write unit test with golang

Unit test can be used to assure the accuracy of a single module. In the pyramid of test field, the wider of unit test coverage, the more complete of its function, the more it can reduce the debug cost of integration test and end-to-end test. In complex system, the longer the task processing link, the higher cost of positioning problem, especially in the problems caused by small module. The following content will share the summary of writting golang unit test cases in PouchContainer.

## 2.1 Table-Driven<span data-type="color" style="color:rgb(36, 41, 46)"><span data-type="background" style="background-color:rgb(255, 255, 255)"> </span></span>Test-DRY

Simple understanding of unit tests is given a certain input of a function, and determine whether can get the expected output. When the tested function has a wide variety of input scene, we can use the Table-Driven to form the organization of test cases, which is shown in the following code. Table-Driven uses array to organize test cases, and proves the validity of functions through the loop execution way.

```go
// from https://golang.org/doc/code.html#Testing
package stringutil

import "testing"

func TestReverse(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"Hello, world", "dlrow ,olleH"},
		{"Hello, 世界", "界世 ,olleH"},
		{"", ""},
	}
	for _, c := range cases {
		got := Reverse(c.in)
		if got != c.want {
			t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
		}
	}
}
```

In order to debug and maintain test cases easily, we can add some auxiliary information to describe current test. For example [reference](https://github.com/alibaba/pouch/blob/master/pkg/reference/parse_test.go#L54) wants to test the input of [punycode](https://en.wikipedia.org/wiki/Punycode) . If there is no `punycode`, the code reviewers or the project maintainer may not know the difference between `xn--bcher-kva.tld/redis:3` and `docker.io/library/redis:3`.

```go
{
		name:  "Normal",
		input: "docker.io/library/nginx:alpine",
		expected: taggedReference{
			Named: namedReference{"docker.io/library/nginx"},
			tag:   "alpine",
		},
		err: nil,
}, {
		name:  "Punycode",
		input: "xn--bcher-kva.tld/redis:3",
		expected: taggedReference{
			Named: namedReference{"xn--bcher-kva.tld/redis"},
			tag:   "3",
		},
		err: nil,
}
```

But behavior of some function is very complex, an input can not be as a complete test cases. Such as [TestTeeReader](https://github.com/golang/go/blob/release-branch.go1.9/src/io/io_test.go#L284) . After TeeReader read <span data-type="color" style="color:rgb(3, 47, 98)"><span data-type="background" style="background-color:rgb(255, 255, 255)"><code>hello, world</code></span></span><span data-type="color" style="color:rgb(3, 47, 98)"><span data-type="background" style="background-color:rgb(255, 255, 255)"> </span></span><span data-type="background" style="background-color:rgb(255, 255, 255)"> from buffer, the data is read out. If read again, it will encounter end-of-file error. This test cases need a single case to complete without the form of Table-Driven.</span>

In simple terms, if you need to copy most of the code when test a function, the test code can be drawn out in the theory, and you can organize test cases with Table-Driven, The principle that we need follow is <strong><span data-type="color" style="color:#F5222D">Don`t Repeat Yourself</span></strong><span data-type="color" style="color:#F5222D"> </span>.

> NOTE: The organization of Table-Driven is recommended by golang community. For more details, please check [here](https://github.com/golang/go/wiki/TableDrivenTests).

## 2.2 Mock - simulate external dependencies

There are also dependence problems during testing, such as the PouchContainer client requiring HTTP server, but this is too heavy for unit test, and it belongs to integration testing. So how do we complete this unit test?

In the world of golang, the implementation of interface belongs to [Duck Type](https://en.wikipedia.org/wiki/Duck_typing) . An interface can be implemented in a variety of ways, as long as the implementation accords the interface definition. If the external constraints are constrained by interface, then these dependence behaviors can be simulated by unit test. The next content will share two common test scenarios.

### 2.2.1 RoundTripper

Let's take the PouchContainer client test as an example. PouchContainer client uses [http.Client](https://golang.org/pkg/net/http/#Client) . And http.Client uses [RoundTripper](https://golang.org/pkg/net/http/#RoundTripper) to perform a HTTP request. It allows developers to customize the logic of sending HTTP requests, which is also an important reason for golang to fully support the HTTP 2 protocol on the original basis.

```plain
http.Client -> http.RoundTripper [http.DefaultTransport]
```

For PouchContainer client, the concerns of test are whether the input destination address is correct or the input query is reasonable, and whether the results can be returned properly. So before the test, the developer needs to make the corresponding RoundTripper implementation ready, which is not responsible for the actual business logic. It is only used to determine whether the input is in line with the expectations.

As the next code shows, PouchContainer `newMockClient` can accept custom request processing logic. In the case of deleting mirror, the developer determines whether the destination address and the HTTP Method are DELETE in the custom logic, so that the function test can be completed without starting HTTP Server.

```go
// https://github.com/alibaba/pouch/blob/master/client/client_mock_test.go#L12-L22
type transportFunc func(*http.Request) (*http.Response, error)

func (transFunc transportFunc) RoundTrip(req *http.Request) (*http.Response, error) {
        return transFunc(req)
}

func newMockClient(handler func(*http.Request) (*http.Response, error)) *http.Client {
        return &http.Client{
                Transport: transportFunc(handler),
        }
}

// https://github.com/alibaba/pouch/blob/master/client/image_remove_test.go
func TestImageRemove(t *testing.T) {
        expectedURL := "/images/image_id"

        httpClient := newMockClient(func(req *http.Request) (*http.Response, error) {
                if !strings.HasPrefix(req.URL.Path, expectedURL) {
                        return nil, fmt.Errorf("expected URL '%s', got '%s'", expectedURL, req.URL)
                }
                if req.Method != "DELETE" {
                        return nil, fmt.Errorf("expected DELETE method, got %s", req.Method)
                }

                return &http.Response{
                        StatusCode: http.StatusNoContent,
                        Body:       ioutil.NopCloser(bytes.NewReader([]byte(""))),
                }, nil
        })

        client := &APIClient{
                HTTPCli: httpClient,
        }

        err := client.ImageRemove(context.Background(), "image_id", false)
        if err != nil {
                t.Fatal(err)
        }
}
```

### 2.2.2 MockImageManager

For the dependence between the internal package, such as the PouchContainer Image API Bridge is dependent on the PouchContainer Daemon ImageManager, and the dependency behavior in it is convened by interface. If we want to test the logic of Image Bridge, we don't have to start containerd, but just need to implement Daemon ImageManager like RoundTripper.

```go
// https://github.com/alibaba/pouch/blob/master/apis/server/image_bridge_test.go
type mockImgePull struct {
        mgr.ImageMgr
        handler func(ctx context.Context, imageRef string, authConfig *types.AuthConfig, out io.Writer) error
}

func (m *mockImgePull) PullImage(ctx context.Context, imageRef string, authConfig *types.AuthConfig, out io.Writer) error {
        return m.handler(ctx, imageRef, authConfig, out)
}

func Test_pullImage_without_tag(t *testing.T) {
        var s Server

        s.ImageMgr = &mockImgePull{
                ImageMgr: &mgr.ImageManager{},
                handler: func(ctx context.Context, imageRef string, authConfig *types.AuthConfig, out io.Writer) error {
                        assert.Equal(t, "reg.abc.com/base/os:7.2", imageRef)
                        return nil
                },
        }
        req := &http.Request{
                Form:   map[string][]string{"fromImage": {"reg.abc.com/base/os:7.2"}},
                Header: map[string][]string{},
        }
        s.pullImage(context.Background(), nil, req)
}
```

### 2.2.3 Summary

Besides the number of functions defined by interfaces are different between ImageManager and RoundTripper, the way of simulation is consistent. Usually, developers can manually define a structure that uses the method as a field, as shown in the next code.

```go
type Do interface {
    Add(x int, y int) int
    Sub(x int, y int) int
}

type mockDo struct {
    addFunc func(x int, y int) int
    subFunc func(x int, y int) int
}

// Add implements Do.Add function.
type (m *mockDo) Add(x int, y int) int {
    return m.addFunc(x, y)
}

// Sub implements Do.Sub function.
type (m *mockDo) Sub(x int, y int) int {
    return m.subFunc(x, y)
}
```

When the interface is larger and more complex, the manual way will bring burden to the developer, so the community provides automatic generation tools to reduce the burden of developers, such as [mockery](https://github.com/vektra/mockery) .

## 2.3 Other branch of learning

Sometimes relying on third party services, the PouchContainer client is a typical case. Duck Type can be used to complete the test. In addition, we can register the http. Handler and start mockHTTPServer to complete the request processing. This is a heavy way to test, and it is recommended that you consider it when you can't pass the Duck Type test, or put it into the integration test.

> NOTE: Someone in the golang community has done it by modifying binary code. [monkeypatch](https://github.com/bouk/monkey). It is recommended that developers design and write testable code instead of using this tool.

## 2.4 Summary

When PouchContainer integrates unit test cases into the code review phase, the reviewer can check how the test cases are running at any time.


# 3. <span data-type="color" style="color:rgb(34, 34, 34)"><span data-type="background" style="background-color:rgb(255, 255, 255)">Summary</span></span>

In the code review phase, code style checks, unit tests, and integration tests should be run in a continuous integration way to help the reviewers make accurate decisions. At present, PouchContainer completes code style check and test operation mainly through TravisCI/CircleCI and  [pouchrobot](https://github.com/pouchcontainer/pouchrobot) .

