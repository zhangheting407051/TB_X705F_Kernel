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

#define LOG_TAG "ValidateHal"

#include "ValidateHal.h"
#include "NeuralNetworks.h"
#include "Utils.h"

#include <android-base/logging.h>

namespace android {
namespace nn {

class MemoryAccessVerifier {
public:
    MemoryAccessVerifier(const hidl_vec<hidl_memory>& pools)
        : mPoolCount(pools.size()), mPoolSizes(mPoolCount) {
        for (size_t i = 0; i < mPoolCount; i++) {
            mPoolSizes[i] = pools[i].size();
        }
    }
    bool validate(const DataLocation& location) {
        if (location.poolIndex >= mPoolCount) {
            LOG(ERROR) << "Invalid poolIndex " << location.poolIndex << "/" << mPoolCount;
            return false;
        }
        const size_t size = mPoolSizes[location.poolIndex];
        // Do the addition using size_t to avoid potential wrap-around problems.
        if (static_cast<size_t>(location.offset) + location.length > size) {
            LOG(ERROR) << "Reference to pool " << location.poolIndex << " with offset "
                       << location.offset << " and length " << location.length
                       << " exceeds pool size of " << size;
            return false;
        }
        return true;
    }

private:
    size_t mPoolCount;
    std::vector<size_t> mPoolSizes;
};

static bool validateOperands(const hidl_vec<Operand>& operands,
                             const hidl_vec<uint8_t>& operandValues,
                             const hidl_vec<hidl_memory>& pools) {
    uint32_t index = 0;
    MemoryAccessVerifier poolVerifier(pools);
    for (auto& operand : operands) {
        // Validate type and dimensions.
        switch (operand.type) {
            case OperandType::FLOAT32:
            case OperandType::INT32:
            case OperandType::UINT32:
            case OperandType::OEM: {
                size_t count = operand.dimensions.size();
                if (count != 0) {
                    LOG(ERROR) << "Operand " << index << ": Scalar data has dimensions of rank "
                               << count;
                    return false;
                }
                break;
            }
            case OperandType::TENSOR_FLOAT32:
            case OperandType::TENSOR_INT32:
            case OperandType::TENSOR_QUANT8_ASYMM:
            case OperandType::TENSOR_OEM_BYTE: {
                if (operand.dimensions.size() == 0) {
                    LOG(ERROR) << "Operand " << index << ": Tensor has dimensions of rank 0";
                    return false;
                }
                break;
            }
            default:
                LOG(ERROR) << "Operand " << index << ": Invalid operand type "
                           << toString(operand.type);
                return false;
        }

        // TODO Validate the numberOfConsumers.
        // TODO Since we have to validate it, there was no point in including it. For the next
        // release, consider removing unless we have an additional process in system space
        // that creates this value. In that case, it would not have to be validated.

        // Validate the scale.
        switch (operand.type) {
            case OperandType::FLOAT32:
            case OperandType::INT32:
            case OperandType::UINT32:
            case OperandType::TENSOR_FLOAT32:
                if (operand.scale != 0.f) {
                    LOG(ERROR) << "Operand " << index << ": Operand of type "
                               << getOperandTypeName(operand.type) << " with a non-zero scale ("
                               << operand.scale;
                    return false;
                }
                break;
            case OperandType::TENSOR_QUANT8_ASYMM:
                if (operand.scale == 0.f) {
                    LOG(ERROR) << "Operand " << index << ": Operand of type "
                               << getOperandTypeName(operand.type) << " with a zero scale";
                    return false;
                }
                break;
            default:
                // No validation for the OEM types. No validation also for TENSOR_INT32,
                // as tensors of this type may be used with or without scale, depending on
                // the operation.
                // TODO We should have had a separate type for TENSOR_INT32 that a scale
                // and those who don't.  Document now and fix in the next release.
                break;
        }

        // Validate the zeroPoint.
        switch (operand.type) {
            case OperandType::FLOAT32:
            case OperandType::INT32:
            case OperandType::UINT32:
            case OperandType::TENSOR_FLOAT32:
            case OperandType::TENSOR_INT32:
                if (operand.zeroPoint != 0) {
                    LOG(ERROR) << "Operand " << index << ": Operand of type "
                               << getOperandTypeName(operand.type) << " with an non-zero zeroPoint "
                               << operand.zeroPoint;
                    return false;
                }
                break;
            default:
                // No validation for the OEM types.
                break;
        }

        // Validate the lifetime and the location.
        const DataLocation& location = operand.location;
        switch (operand.lifetime) {
            case OperandLifeTime::CONSTANT_COPY:
                if (location.poolIndex != 0) {
                    LOG(ERROR) << "Operand " << index
                               << ": CONSTANT_COPY with a non-zero poolIndex "
                               << location.poolIndex;
                    return false;
                }
                // Do the addition using size_t to avoid potential wrap-around problems.
                if (static_cast<size_t>(location.offset) + location.length > operandValues.size()) {
                    LOG(ERROR) << "Operand " << index
                               << ": OperandValue location out of range.  Starts at "
                               << location.offset << ", length " << location.length << ", max "
                               << operandValues.size();
                    return false;
                }
                break;
            case OperandLifeTime::CONSTANT_REFERENCE:
                if (!poolVerifier.validate(location)) {
                    return false;
                }
                break;
            case OperandLifeTime::TEMPORARY_VARIABLE:
            case OperandLifeTime::MODEL_INPUT:
            case OperandLifeTime::MODEL_OUTPUT:
            case OperandLifeTime::NO_VALUE:
                if (location.poolIndex != 0 || location.offset != 0 || location.length != 0) {
                    LOG(ERROR) << "Operand " << index << ": Unexpected poolIndex "
                               << location.poolIndex << ", offset " << location.offset
                               << ", or length " << location.length << " for operand of lifetime "
                               << toString(operand.lifetime);
                    return false;
                }
                break;
            default:
                LOG(ERROR) << "Operand " << index << ": Invalid lifetime "
                           << toString(operand.lifetime);
                return false;
        }

        // For constants, validate that the length is as expected. The other lifetimes
        // expect the length to be 0. Don't validate for OEM types.
        if (operand.lifetime == OperandLifeTime::CONSTANT_REFERENCE ||
            operand.lifetime == OperandLifeTime::CONSTANT_COPY) {
            if (operand.type != OperandType::OEM &&
                operand.type != OperandType::TENSOR_OEM_BYTE) {
                uint32_t expectedLength = sizeOfData(operand.type, operand.dimensions);
                if (location.length != expectedLength) {
                    LOG(ERROR) << "Operand " << index << ": For operand " << toString(operand)
                               << " expected a size of " << expectedLength << " but got "
                               << location.length;
                    return false;
                }
            }
        }

        index++;
    }
    return true;
}

static bool validateOperations(const hidl_vec<Operation>& operations,
                               const hidl_vec<Operand>& operands) {
    const size_t operandCount = operands.size();
    // This vector keeps track of whether there's an operation that writes to
    // each operand. It is used to validate that temporary variables and
    // model outputs will be written to.
    std::vector<bool> writtenTo(operandCount, false);
    for (auto& op : operations) {
        if (!validCode(kNumberOfOperationTypes, kNumberOfOperationTypesOEM,
                       static_cast<uint32_t>(op.type))) {
            LOG(ERROR) << "Invalid operation type " << toString(op.type);
            return false;
        }
        // TODO Validate that the number of inputs and outputs, and their types, is correct
        // for the operation. This is currently done in CpuExecutor but should be done
        // here for all drivers.
        for (uint32_t i : op.inputs) {
            if (i >= operandCount) {
                LOG(ERROR) << "Operation input index out of range " << i << "/" << operandCount;
                return false;
            }
        }
        for (uint32_t i : op.outputs) {
            if (i >= operandCount) {
                LOG(ERROR) << "Operation output index out of range " << i << "/" << operandCount;
                return false;
            }
            const Operand& operand = operands[i];
            if (operand.lifetime != OperandLifeTime::TEMPORARY_VARIABLE &&
                operand.lifetime != OperandLifeTime::MODEL_OUTPUT) {
                LOG(ERROR) << "Writing to an operand with incompatible lifetime "
                           << toString(operand.lifetime);
                return false;
            }

            // Check that we only write once to an operand.
            if (writtenTo[i]) {
                LOG(ERROR) << "Operand " << i << " written a second time";
                return false;
            }
            writtenTo[i] = true;
        }
    }
    for (size_t i = 0; i < operandCount; i++) {
        if (!writtenTo[i]) {
            const Operand& operand = operands[i];
            if (operand.lifetime == OperandLifeTime::TEMPORARY_VARIABLE ||
                operand.lifetime == OperandLifeTime::MODEL_OUTPUT) {
                LOG(ERROR) << "Operand " << i << " with lifetime " << toString(operand.lifetime)
                           << " is not being written to.";
                return false;
            }
        }
    }
    // TODO More whole graph verifications are possible, for example that an
    // operand is not use as input & output for the same op, and more
    // generally that it is acyclic.
    return true;
}

static bool validatePools(const hidl_vec<hidl_memory>& pools) {
    for (const hidl_memory& memory : pools) {
        const auto name = memory.name();
        if (name != "ashmem" && name != "mmap_fd") {
            LOG(ERROR) << "Unsupported memory type " << name;
            return false;
        }
        if (memory.handle() == nullptr) {
            LOG(ERROR) << "Memory of type " << name << " is null";
            return false;
        }
    }
    return true;
}

static bool validateModelInputOutputs(const hidl_vec<uint32_t> indexes,
                                      const hidl_vec<Operand>& operands, OperandLifeTime lifetime) {
    const size_t operandCount = operands.size();
    for (uint32_t i : indexes) {
        if (i >= operandCount) {
            LOG(ERROR) << "Model input or output index out of range " << i << "/" << operandCount;
            return false;
        }
        const Operand& operand = operands[i];
        if (operand.lifetime != lifetime) {
            LOG(ERROR) << "Model input or output has lifetime of " << toString(operand.lifetime)
                       << " instead of the expected " << toString(lifetime);
            return false;
        }
    }
    return true;
}

bool validateModel(const Model& model) {
    return (validateOperands(model.operands, model.operandValues, model.pools) &&
            validateOperations(model.operations, model.operands) &&
            validateModelInputOutputs(model.inputIndexes, model.operands,
                                      OperandLifeTime::MODEL_INPUT) &&
            validateModelInputOutputs(model.outputIndexes, model.operands,
                                      OperandLifeTime::MODEL_OUTPUT) &&
            validatePools(model.pools));
}

// Validates the arguments of a request. type is either "input" or "output" and is used
// for printing error messages. The operandIndexes is the appropriate array of input
// or output operand indexes that was passed to the ANeuralNetworksModel_identifyInputsAndOutputs.
static bool validateRequestArguments(const hidl_vec<RequestArgument>& requestArguments,
                                     const hidl_vec<uint32_t>& operandIndexes,
                                     const hidl_vec<Operand>& operands,
                                     const hidl_vec<hidl_memory>& pools, const char* type) {
    MemoryAccessVerifier poolVerifier(pools);
    // The request should specify as many arguments as were described in the model.
    const size_t requestArgumentCount = requestArguments.size();
    if (requestArgumentCount != operandIndexes.size()) {
        LOG(ERROR) << "Request specifies " << requestArgumentCount << " " << type
                   << "s but the model has " << operandIndexes.size();
        return false;
    }
    for (size_t requestArgumentIndex = 0; requestArgumentIndex < requestArgumentCount;
         requestArgumentIndex++) {
        const RequestArgument& requestArgument = requestArguments[requestArgumentIndex];
        const DataLocation& location = requestArgument.location;
        // Get the operand index for this argument. We extract it from the list
        // that was provided in the call to ANeuralNetworksModel_identifyInputsAndOutputs.
        // We assume in this function that the model has been validated already.
        const uint32_t operandIndex = operandIndexes[requestArgumentIndex];
        const Operand& operand = operands[operandIndex];
        if (requestArgument.hasNoValue) {
            if (location.poolIndex != 0 || location.offset != 0 || location.length != 0 ||
                requestArgument.dimensions.size() != 0) {
                LOG(ERROR) << "Request " << type << " " << requestArgumentIndex
                           << " has no value yet has details.";
                return false;
            }
        } else {
            // Validate the location.
            if (!poolVerifier.validate(location)) {
                return false;
            }
            // If the argument specified a dimension, validate it.
            uint32_t rank = requestArgument.dimensions.size();
            if (rank == 0) {
                // Validate that all the dimensions are specified in the model.
                for (size_t i = 0; i < operand.dimensions.size(); i++) {
                    if (operand.dimensions[i] == 0) {
                        LOG(ERROR) << "Model has dimension " << i
                                   << " set to 0 but the request does specify the dimension.";
                        return false;
                    }
                }
            } else {
                if (rank != operand.dimensions.size()) {
                    LOG(ERROR) << "Request " << type << " " << requestArgumentIndex
                               << " has number of dimensions (" << rank
                               << ") different than the model's (" << operand.dimensions.size()
                               << ")";
                    return false;
                }
                for (size_t i = 0; i < rank; i++) {
                    if (requestArgument.dimensions[i] != operand.dimensions[i] &&
                        operand.dimensions[i] != 0) {
                        LOG(ERROR) << "Request " << type << " " << requestArgumentIndex
                                   << " has dimension " << i << " of "
                                   << requestArgument.dimensions[i]
                                   << " different than the model's " << operand.dimensions[i];
                        return false;
                    }
                    if (requestArgument.dimensions[i] == 0) {
                        LOG(ERROR) << "Request " << type << " " << requestArgumentIndex
                                   << " has dimension " << i << " of zero";
                        return false;
                    }
                }
            }
        }
    }
    return true;
}

bool validateRequest(const Request& request, const Model& model) {
    return (validateRequestArguments(request.inputs, model.inputIndexes, model.operands,
                                     request.pools, "input") &&
            validateRequestArguments(request.outputs, model.outputIndexes, model.operands,
                                     request.pools, "output") &&
            validatePools(request.pools));
}

}  // namespace nn
}  // namespace android
