/*
 * Copyright (C) 2017 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

#include "NeuralNetworksWrapper.h"

#include "Memory.h"

#include <android/sharedmem.h>
#include <fstream>
#include <gtest/gtest.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <unistd.h>

using WrapperCompilation = ::android::nn::wrapper::Compilation;
using WrapperExecution = ::android::nn::wrapper::Execution;
using WrapperMemory = ::android::nn::wrapper::Memory;
using WrapperModel = ::android::nn::wrapper::Model;
using WrapperOperandType = ::android::nn::wrapper::OperandType;
using WrapperResult = ::android::nn::wrapper::Result;
using WrapperType = ::android::nn::wrapper::Type;

namespace {

typedef float Matrix3x4[3][4];

// Tests the various ways to pass weights and input/output data.
class MemoryTest : public ::testing::Test {
protected:
    void SetUp() override {}

    const Matrix3x4 matrix1 = {{1.f, 2.f, 3.f, 4.f}, {5.f, 6.f, 7.f, 8.f}, {9.f, 10.f, 11.f, 12.f}};
    const Matrix3x4 matrix2 = {{100.f, 200.f, 300.f, 400.f},
                               {500.f, 600.f, 700.f, 800.f},
                               {900.f, 1000.f, 1100.f, 1200.f}};
    const Matrix3x4 matrix3 = {{20.f, 30.f, 40.f, 50.f},
                               {21.f, 22.f, 23.f, 24.f},
                               {31.f, 32.f, 33.f, 34.f}};
    const Matrix3x4 expected3 = {{121.f, 232.f, 343.f, 454.f},
                                 {526.f, 628.f, 730.f, 832.f},
                                 {940.f, 1042.f, 1144.f, 1246.f}};
};

// Tests to ensure that various kinds of memory leaks do not occur.
class MemoryLeakTest : public MemoryTest {
protected:
    void SetUp() override;

    int mMaxMapCount = 0;
};

void MemoryLeakTest::SetUp() {
    std::ifstream maxMapCountStream("/proc/sys/vm/max_map_count");
    if (maxMapCountStream) {
        maxMapCountStream >> mMaxMapCount;
    }
}

// Check that the values are the same. This works only if dealing with integer
// value, otherwise we should accept values that are similar if not exact.
int CompareMatrices(const Matrix3x4& expected, const Matrix3x4& actual) {
    int errors = 0;
    for (int i = 0; i < 3; i++) {
        for (int j = 0; j < 4; j++) {
            if (expected[i][j] != actual[i][j]) {
                printf("expected[%d][%d] != actual[%d][%d], %f != %f\n", i, j, i, j,
                       static_cast<double>(expected[i][j]), static_cast<double>(actual[i][j]));
                errors++;
            }
        }
    }
    return errors;
}

// As well as serving as a functional test for ASharedMemory, also
// serves as a regression test for http://b/69685100 "RunTimePoolInfo
// leaks shared memory regions".
//
// TODO: test non-zero offset.
TEST_F(MemoryLeakTest, TestASharedMemory) {
    ASSERT_GT(mMaxMapCount, 0);

    // Layout where to place matrix2 and matrix3 in the memory we'll allocate.
    // We have gaps to test that we don't assume contiguity.
    constexpr uint32_t offsetForMatrix2 = 20;
    constexpr uint32_t offsetForMatrix3 = offsetForMatrix2 + sizeof(matrix2) + 30;
    constexpr uint32_t memorySize = offsetForMatrix3 + sizeof(matrix3) + 60;

    int weightsFd = ASharedMemory_create("weights", memorySize);
    ASSERT_GT(weightsFd, -1);
    uint8_t* weightsData = (uint8_t*)mmap(nullptr, memorySize, PROT_READ | PROT_WRITE,
                                          MAP_SHARED, weightsFd, 0);
    ASSERT_NE(weightsData, nullptr);
    memcpy(weightsData + offsetForMatrix2, matrix2, sizeof(matrix2));
    memcpy(weightsData + offsetForMatrix3, matrix3, sizeof(matrix3));
    WrapperMemory weights(memorySize, PROT_READ | PROT_WRITE, weightsFd, 0);
    ASSERT_TRUE(weights.isValid());

    WrapperModel model;
    WrapperOperandType matrixType(WrapperType::TENSOR_FLOAT32, {3, 4});
    WrapperOperandType scalarType(WrapperType::INT32, {});
    int32_t activation(0);
    auto a = model.addOperand(&matrixType);
    auto b = model.addOperand(&matrixType);
    auto c = model.addOperand(&matrixType);
    auto d = model.addOperand(&matrixType);
    auto e = model.addOperand(&matrixType);
    auto f = model.addOperand(&scalarType);

    model.setOperandValueFromMemory(e, &weights, offsetForMatrix2, sizeof(Matrix3x4));
    model.setOperandValueFromMemory(a, &weights, offsetForMatrix3, sizeof(Matrix3x4));
    model.setOperandValue(f, &activation, sizeof(activation));
    model.addOperation(ANEURALNETWORKS_ADD, {a, c, f}, {b});
    model.addOperation(ANEURALNETWORKS_ADD, {b, e, f}, {d});
    model.identifyInputsAndOutputs({c}, {d});
    ASSERT_TRUE(model.isValid());
    model.finish();

    // Test the two node model.
    constexpr uint32_t offsetForMatrix1 = 20;
    int inputFd = ASharedMemory_create("input", offsetForMatrix1 + sizeof(Matrix3x4));
    ASSERT_GT(inputFd, -1);
    uint8_t* inputData = (uint8_t*)mmap(nullptr, offsetForMatrix1 + sizeof(Matrix3x4),
                                        PROT_READ | PROT_WRITE, MAP_SHARED, inputFd, 0);
    ASSERT_NE(inputData, nullptr);
    memcpy(inputData + offsetForMatrix1, matrix1, sizeof(Matrix3x4));
    WrapperMemory input(offsetForMatrix1 + sizeof(Matrix3x4), PROT_READ, inputFd, 0);
    ASSERT_TRUE(input.isValid());

    constexpr uint32_t offsetForActual = 32;
    int outputFd = ASharedMemory_create("output", offsetForActual + sizeof(Matrix3x4));
    ASSERT_GT(outputFd, -1);
    uint8_t* outputData = (uint8_t*)mmap(nullptr, offsetForActual + sizeof(Matrix3x4),
                                         PROT_READ | PROT_WRITE, MAP_SHARED, outputFd, 0);
    ASSERT_NE(outputData, nullptr);
    memset(outputData, 0, offsetForActual + sizeof(Matrix3x4));
    WrapperMemory actual(offsetForActual + sizeof(Matrix3x4), PROT_READ | PROT_WRITE, outputFd, 0);
    ASSERT_TRUE(actual.isValid());

    WrapperCompilation compilation2(&model);
    ASSERT_EQ(compilation2.finish(), WrapperResult::NO_ERROR);

    for (int i = 0, e = mMaxMapCount + 10; i < e; i++) {
        SCOPED_TRACE(i);
        WrapperExecution execution2(&compilation2);
        ASSERT_EQ(execution2.setInputFromMemory(0, &input, offsetForMatrix1, sizeof(Matrix3x4)),
                  WrapperResult::NO_ERROR);
        ASSERT_EQ(execution2.setOutputFromMemory(0, &actual, offsetForActual, sizeof(Matrix3x4)),
                  WrapperResult::NO_ERROR);
        ASSERT_EQ(execution2.compute(), WrapperResult::NO_ERROR);
        ASSERT_EQ(CompareMatrices(expected3,
                                  *reinterpret_cast<Matrix3x4*>(outputData + offsetForActual)), 0);
    }

    close(weightsFd);
    close(inputFd);
    close(outputFd);
}

TEST_F(MemoryTest, TestFd) {
    // Create a file that contains matrix2 and matrix3.
    char path[] = "/data/local/tmp/TestMemoryXXXXXX";
    int fd = mkstemp(path);
    const uint32_t offsetForMatrix2 = 20;
    const uint32_t offsetForMatrix3 = 200;
    static_assert(offsetForMatrix2 + sizeof(matrix2) < offsetForMatrix3, "matrices overlap");
    lseek(fd, offsetForMatrix2, SEEK_SET);
    write(fd, matrix2, sizeof(matrix2));
    lseek(fd, offsetForMatrix3, SEEK_SET);
    write(fd, matrix3, sizeof(matrix3));
    fsync(fd);

    WrapperMemory weights(offsetForMatrix3 + sizeof(matrix3), PROT_READ, fd, 0);
    ASSERT_TRUE(weights.isValid());

    WrapperModel model;
    WrapperOperandType matrixType(WrapperType::TENSOR_FLOAT32, {3, 4});
    WrapperOperandType scalarType(WrapperType::INT32, {});
    int32_t activation(0);
    auto a = model.addOperand(&matrixType);
    auto b = model.addOperand(&matrixType);
    auto c = model.addOperand(&matrixType);
    auto d = model.addOperand(&matrixType);
    auto e = model.addOperand(&matrixType);
    auto f = model.addOperand(&scalarType);

    model.setOperandValueFromMemory(e, &weights, offsetForMatrix2, sizeof(Matrix3x4));
    model.setOperandValueFromMemory(a, &weights, offsetForMatrix3, sizeof(Matrix3x4));
    model.setOperandValue(f, &activation, sizeof(activation));
    model.addOperation(ANEURALNETWORKS_ADD, {a, c, f}, {b});
    model.addOperation(ANEURALNETWORKS_ADD, {b, e, f}, {d});
    model.identifyInputsAndOutputs({c}, {d});
    ASSERT_TRUE(model.isValid());
    model.finish();

    // Test the three node model.
    Matrix3x4 actual;
    memset(&actual, 0, sizeof(actual));
    WrapperCompilation compilation2(&model);
    ASSERT_EQ(compilation2.finish(), WrapperResult::NO_ERROR);
    WrapperExecution execution2(&compilation2);
    ASSERT_EQ(execution2.setInput(0, matrix1, sizeof(Matrix3x4)), WrapperResult::NO_ERROR);
    ASSERT_EQ(execution2.setOutput(0, actual, sizeof(Matrix3x4)), WrapperResult::NO_ERROR);
    ASSERT_EQ(execution2.compute(), WrapperResult::NO_ERROR);
    ASSERT_EQ(CompareMatrices(expected3, actual), 0);

    close(fd);
    unlink(path);
}

// Regression test for http://b/69621433 "MemoryFd leaks shared memory regions".
TEST_F(MemoryLeakTest, IterativelyGetPointer) {
    ASSERT_GT(mMaxMapCount, 0);

    static const size_t size = 1;
    const int iterations = mMaxMapCount + 10;

    int fd = ASharedMemory_create(nullptr, size);
    ASSERT_GE(fd, 0);

    uint8_t* buf = (uint8_t*)mmap(nullptr, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    ASSERT_NE(buf, nullptr);
    *buf = 0;

    {
        // Scope "mem" in such a way that any shared memory regions it
        // owns will be released before we check the value of *buf: We
        // want to verify that the explicit mmap() above is not
        // perturbed by any mmap()/munmap() that results from methods
        // invoked on "mem".

        WrapperMemory mem(size, PROT_READ | PROT_WRITE, fd, 0);
        ASSERT_TRUE(mem.isValid());

        auto internalMem = reinterpret_cast<::android::nn::Memory*>(mem.get());
        uint8_t *dummy;
        for (int i = 0; i < iterations; i++) {
            SCOPED_TRACE(i);
            ASSERT_EQ(internalMem->getPointer(&dummy), ANEURALNETWORKS_NO_ERROR);
            (*dummy)++;
        }
    }

    ASSERT_EQ(*buf, (uint8_t)iterations);

    ASSERT_EQ(munmap(buf, size), 0);

    close(fd);
}

// Regression test for http://b/69621433 "MemoryFd leaks shared memory regions".
TEST_F(MemoryLeakTest, IterativelyInstantiate) {
    ASSERT_GT(mMaxMapCount, 0);

    for (int i = 0, e = mMaxMapCount + 10; i < e; i++) {
        SCOPED_TRACE(i);

        static const size_t size = 1;
        int fd = ASharedMemory_create(nullptr, size);
        ASSERT_GE(fd, 0);
        WrapperMemory mem(size, PROT_READ | PROT_WRITE, fd, 0);
        ASSERT_TRUE(mem.isValid());

        auto internalMem = reinterpret_cast<::android::nn::Memory*>(mem.get());
        uint8_t *dummy;
        ASSERT_EQ(internalMem->getPointer(&dummy), ANEURALNETWORKS_NO_ERROR);

        close(fd);
    }
}

}  // end namespace
