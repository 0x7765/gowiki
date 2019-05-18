# zap支持日志归档

zap默认不支持文件归档，官方推荐的是 [lumberjack](https://github.com/natefinch/lumberjack)这个库

# 使用

```go
const (
	logpath = "app.log"
)

var (
	logger *zap.Logger
)

func initLog1() {
	hook := lumberjack.Logger{
		Filename:   logpath, // 日志文件路径
		MaxSize:    1024,    // megabytes
		MaxBackups: 3,       // 最多保留3个备份
		MaxAge:     7,       // days
		Compress:   true,    // 是否压缩 disabled by default
	}
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	core := zapcore.NewCore(
		zapcore.NewConsoleEncoder(encoderConfig),
		zapcore.AddSync(&hook),
		zapcore.DebugLevel,
	)
	logger = zap.New(core)
	logger.Info("DefaultLogger init1 success")
}
```

# 日志输出多个地址

同一份日志，输出到kafka、文件、console

```go
type KafkaLogger struct {
}

func (KafkaLogger) Write(p []byte) (n int, err error) {
	fmt.Println("mock write to kafka...", string(p))
	return len(p), nil
}

func (KafkaLogger) Sync() error {
	return nil
}

func initLog2() {

	hook := lumberjack.Logger{
		Filename:   logpath, // 日志文件路径
		MaxSize:    1024,    // megabytes
		MaxBackups: 3,       // 最多保留3个备份
		MaxAge:     7,       // days
		Compress:   true,    // 是否压缩 disabled by default
	}

	jsonEncoder := zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
	consoleEncoder := zapcore.NewConsoleEncoder(zap.NewDevelopmentEncoderConfig())

	core := zapcore.NewTee(
		// 打印在kafka topic中（伪造的case）
		zapcore.NewCore(jsonEncoder, KafkaLogger{}, zap.InfoLevel),
		// 打印在控制台
		zapcore.NewCore(consoleEncoder, os.Stderr, zap.InfoLevel),
		// 打印在文件中
		zapcore.NewCore(consoleEncoder, zapcore.AddSync(&hook), zap.InfoLevel),
	)
	logger = zap.New(core)
	logger.Info("DefaultLogger init2 success")
}
```